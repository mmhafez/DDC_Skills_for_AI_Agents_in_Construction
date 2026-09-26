---
name: "p6-schedule-exporter"
description: "Export construction schedules to Primavera P6 importable formats - P6 XML and XER. Convert Excel/CSV or DDC schedule data into P6-loadable files with WBS, activities, relationships (FS/SS/FF/SF), lags, calendars, resources and cost assignments."
homepage: "https://datadrivenconstruction.io"
metadata: {"openclaw": {"emoji": "📅", "os": ["darwin", "linux", "win32"], "homepage": "https://datadrivenconstruction.io", "requires": {"bins": ["python3"]}}}
---
# Primavera P6 Schedule Exporter

## Business Case

### Problem Statement
The DDC scheduling skills can plan, cost-load, level and analyse a schedule, and the `xml-reader`
skill can **read** Primavera P6 XML/XER - but until now nothing in the collection could **write**
a file that P6 can open. The result:

- Planners re-key activity logic into P6 by hand, which is slow and error-prone.
- Client/GC handover often mandates a native P6 schedule.
- Analysis done in `critical-path-analyzer` never flows back into the tool of record.

### Solution
Take the *same* activity + predecessor table produced by `critical-path-analyzer`,
`cwicr-schedule-integrator`, `bim-to-schedule-4d` or `oce-scheduling-4d5d` and serialise it to
Primavera P6. The collection becomes round-trip capable: read P6, analyse it, write it back.

### What it produces

| Format | Use it for | Content |
|--------|-----------|---------|
| **P6 XML** | P6 8.x-19.x+, EPPM, most reliable path | Project, Calendar, WBS, Activities, Relationships, Resources, Resource Assignments |
| **XER** | Older P6 releases and tools that only ingest XER | `PROJECT`, `CALENDAR`, `PROJWBS`, `TASK`, `RSRC`, `TASKRSRC`, `TASKPRED` tables |
| **Excel (P6 layout)** | P6's Excel import wizard, or manual review | `Activities`, `Relationships`, `WBS`, `Resources` sheets |

### Capabilities
- Activities, milestones, WBS hierarchy, 5-day work-week calendar.
- Relationships: **FS / SS / FF / SF** with lag in days.
- Planned dates forward-passed from logic (working-day aware).
- Percent complete, actual start/finish, notes, primary constraints.
- Resources and cost-loadable resource assignments.
- Validation (dangling logic, cycles, duplicate IDs, bad durations/percentages) **before** export.

## Technical Implementation

The four blocks below form one module. Save them together as `p6_schedule_exporter.py`.

### 1. Schedule model, import, logic and validation

```python
"""
P6 Schedule Exporter - export construction schedules to Primavera P6 formats.

Reads the same activity/predecessor tables produced by the DDC scheduling skills
(`critical-path-analyzer`, `cwicr-schedule-integrator`, `bim-to-schedule-4d`,
`oce-scheduling-4d5d`) and writes Primavera P6-importable output:

* P6 XML  - APIBusinessObjects schema (Project, Calendar, WBS, Activity,
            Resource, ResourceAssignment, Relationship)
* XER     - tab-delimited PROJECT / CALENDAR / PROJWBS / TASK / RSRC /
            TASKRSRC / TASKPRED
* Excel   - flat activity + relationship layout for the P6 Excel import wizard
"""
from __future__ import annotations

import re
import xml.etree.ElementTree as ET
from collections import OrderedDict, defaultdict, deque
from dataclasses import dataclass
from datetime import date, datetime, timedelta
from enum import Enum
from pathlib import Path
from typing import Any, Dict, Iterable, List, Optional, Tuple, Union

import pandas as pd

XSI_NS = "http://www.w3.org/2001/XMLSchema-instance"


# ---------------------------------------------------------------------------
# Enums
# ---------------------------------------------------------------------------
class DependencyType(Enum):
    """Activity relationship types, P6 names and XER codes."""

    FS = "FS"
    SS = "SS"
    FF = "FF"
    SF = "SF"

    @property
    def p6_xml_label(self) -> str:
        return {
            "FS": "Finish to Start",
            "SS": "Start to Start",
            "FF": "Finish to Finish",
            "SF": "Start to Finish",
        }[self.value]

    @property
    def xer_code(self) -> str:
        return {"FS": "PR_FS", "SS": "PR_SS", "FF": "PR_FF", "SF": "PR_SF"}[self.value]

    @classmethod
    def parse(cls, value: "Union[str, DependencyType]") -> "DependencyType":
        if isinstance(value, DependencyType):
            return value
        token = str(value).strip().upper().replace(" ", "").replace("_", "")
        aliases = {
            "FS": cls.FS, "FINISHTOSTART": cls.FS, "PRFS": cls.FS,
            "SS": cls.SS, "STARTTOSTART": cls.SS, "PRSS": cls.SS,
            "FF": cls.FF, "FINISHTOFINISH": cls.FF, "PRFF": cls.FF,
            "SF": cls.SF, "STARTTOFINISH": cls.SF, "PRSF": cls.SF,
        }
        if token not in aliases:
            raise ValueError(f"Unknown dependency type: {value!r}")
        return aliases[token]


class ActivityType(Enum):
    TASK_DEPENDENT = "Task Dependent"
    RESOURCE_DEPENDENT = "Resource Dependent"
    START_MILESTONE = "Start Milestone"
    FINISH_MILESTONE = "Finish Milestone"
    LEVEL_OF_EFFORT = "Level of Effort"
    WBS_SUMMARY = "WBS Summary"

    @property
    def xer_code(self) -> str:
        return {
            "Task Dependent": "TT_Task",
            "Resource Dependent": "TT_Rsrc",
            "Start Milestone": "TT_Mile",
            "Finish Milestone": "TT_FinMile",
            "Level of Effort": "TT_LOE",
            "WBS Summary": "TT_WBS",
        }[self.value]

    @property
    def is_milestone(self) -> bool:
        return self in (ActivityType.START_MILESTONE, ActivityType.FINISH_MILESTONE)

    @classmethod
    def parse(cls, value: "Union[str, ActivityType]") -> "ActivityType":
        if isinstance(value, ActivityType):
            return value
        token = str(value).strip().lower()
        aliases = {
            "task": cls.TASK_DEPENDENT, "task dependent": cls.TASK_DEPENDENT,
            "tt_task": cls.TASK_DEPENDENT, "task_dependent": cls.TASK_DEPENDENT,
            "resource": cls.RESOURCE_DEPENDENT, "resource dependent": cls.RESOURCE_DEPENDENT,
            "tt_rsrc": cls.RESOURCE_DEPENDENT,
            "start milestone": cls.START_MILESTONE, "milestone": cls.START_MILESTONE,
            "start_milestone": cls.START_MILESTONE, "tt_mile": cls.START_MILESTONE,
            "finish milestone": cls.FINISH_MILESTONE, "finish_milestone": cls.FINISH_MILESTONE,
            "tt_finmile": cls.FINISH_MILESTONE,
            "level of effort": cls.LEVEL_OF_EFFORT, "loe": cls.LEVEL_OF_EFFORT,
            "tt_loe": cls.LEVEL_OF_EFFORT,
            "wbs summary": cls.WBS_SUMMARY, "wbs_summary": cls.WBS_SUMMARY, "tt_wbs": cls.WBS_SUMMARY,
        }
        if token not in aliases:
            raise ValueError(f"Unknown activity type: {value!r}")
        return aliases[token]


# ---------------------------------------------------------------------------
# Working-day calendar
# ---------------------------------------------------------------------------
class WorkCalendar:
    """Minimal working-day calendar (Mon-Fri by default)."""

    def __init__(self, workdays: Iterable[int] = (0, 1, 2, 3, 4),
                 holidays: Optional[Iterable[date]] = None,
                 hours_per_day: float = 8.0):
        self.workdays = set(workdays)
        self.holidays = set(holidays or [])
        self.hours_per_day = float(hours_per_day)

    def is_working_day(self, d: date) -> bool:
        return d.weekday() in self.workdays and d not in self.holidays

    def snap_forward(self, d: date) -> date:
        while not self.is_working_day(d):
            d += timedelta(days=1)
        return d

    def snap_backward(self, d: date) -> date:
        while not self.is_working_day(d):
            d -= timedelta(days=1)
        return d

    def add_working_days(self, start: date, working_days: int) -> date:
        """Advance (or, for negative values, retreat) by whole working days."""
        working_days = int(round(working_days))
        if working_days < 0:
            return self.subtract_working_days(start, -working_days)
        current = self.snap_forward(start)
        remaining = working_days
        while remaining > 0:
            current += timedelta(days=1)
            if self.is_working_day(current):
                remaining -= 1
        return current

    def subtract_working_days(self, start: date, working_days: int) -> date:
        working_days = int(round(working_days))
        if working_days < 0:
            return self.add_working_days(start, -working_days)
        current = self.snap_backward(start)
        remaining = working_days
        while remaining > 0:
            current -= timedelta(days=1)
            if self.is_working_day(current):
                remaining -= 1
        return current


# ---------------------------------------------------------------------------
# Data model
# ---------------------------------------------------------------------------
@dataclass
class P6Activity:
    activity_id: str
    name: str
    duration_days: float = 0.0
    activity_type: ActivityType = ActivityType.TASK_DEPENDENT
    wbs: str = "Project"
    start: Optional[date] = None
    finish: Optional[date] = None
    actual_start: Optional[date] = None
    actual_finish: Optional[date] = None
    percent_complete: float = 0.0
    constraint_type: Optional[str] = None
    constraint_date: Optional[date] = None
    notes: str = ""
    object_id: int = 0
    computed_start: Optional[date] = None
    computed_finish: Optional[date] = None

    def planned_start(self) -> Optional[date]:
        return self.computed_start or self.start

    def planned_finish(self) -> Optional[date]:
        return self.computed_finish or self.finish

    @property
    def is_milestone(self) -> bool:
        return self.activity_type.is_milestone


@dataclass
class P6Relationship:
    predecessor: str
    successor: str
    dep_type: DependencyType = DependencyType.FS
    lag_days: float = 0.0
    comments: str = ""
    object_id: int = 0


@dataclass
class P6Resource:
    resource_id: str
    name: str
    resource_type: str = "Labor"
    unit: str = "h"
    object_id: int = 0

    @property
    def xer_type(self) -> str:
        # P6 XER resource-type codes. Note the non-obvious spellings: Non-Labor is
        # RT_Equip and Material is RT_Mat (RT_NonLabor / RT_Material are not valid P6 codes).
        return {"Labor": "RT_Labor", "Nonlabor": "RT_Equip", "Material": "RT_Mat"}.get(
            self.resource_type, "RT_Labor")


@dataclass
class P6Assignment:
    activity_id: str
    resource_id: str
    planned_units: float = 0.0
    cost_per_unit: float = 0.0
    object_id: int = 0

    @property
    def planned_cost(self) -> float:
        return self.planned_units * self.cost_per_unit


@dataclass
class ValidationIssue:
    severity: str  # "error" | "warning"
    code: str
    message: str
    activity_id: str = ""

    def __str__(self) -> str:
        where = f" [{self.activity_id}]" if self.activity_id else ""
        return f"{self.severity.upper()} {self.code}{where}: {self.message}"


# ---------------------------------------------------------------------------
# Schedule
# ---------------------------------------------------------------------------
# Predecessor tokens look like: "A100", "A100 FS +3", "A100:SS-2", "A100+5"
_PRED_SPACE_RE = re.compile(
    r"^(?P<id>.+?)\s+(?P<type>FS|SS|FF|SF|PR_FS|PR_SS|PR_FF|PR_SF)\s*"
    r"(?P<lag>[+-]?\d+(?:\.\d+)?)?$", re.IGNORECASE)
_PRED_COLON_RE = re.compile(
    r"^(?P<id>.+?)\s*:\s*(?P<type>[A-Za-z_]+)?\s*(?P<lag>[+-]?\d+(?:\.\d+)?)?$")


class P6Schedule:
    """In-memory schedule that serializes to Primavera P6."""

    COLUMN_ALIASES: Dict[str, List[str]] = {
        "activity_id": ["activity_id", "activity id", "activityid", "task_code",
                        "task code", "id", "activity", "task id"],
        "name": ["name", "activity_name", "activity name", "task_name", "task name",
                 "description", "title"],
        "duration": ["duration", "duration_days", "duration (days)", "planned_duration",
                     "original_duration", "days", "target_drtn_hr_cnt"],
        "predecessors": ["predecessors", "predecessor", "predecessor_ids",
                         "predecessors_list", "pred", "predecessor(s)"],
        "wbs": ["wbs", "wbs_path", "wbs_code", "wbs code", "section", "phase", "wbs name"],
        "start": ["start", "start_date", "planned_start", "early_start",
                  "planned start", "early start", "target_start_date"],
        "finish": ["finish", "finish_date", "planned_finish", "early_finish",
                   "planned finish", "early finish", "target_end_date"],
        "percent_complete": ["percent_complete", "% complete", "%complete", "progress",
                             "complete", "phys_complete_pct"],
        "activity_type": ["activity_type", "activity type", "type", "task_type"],
        "actual_start": ["actual_start", "actual_start_date", "act_start_date"],
        "actual_finish": ["actual_finish", "actual_finish_date", "act_end_date"],
        "resources": ["resources", "resource", "resource_names", "resource ids"],
        "constraint_type": ["constraint_type", "constraint type", "constraint", "cstr_type"],
        "constraint_date": ["constraint_date", "constraint date", "cstr_date"],
    }

    def __init__(self, project_id: str = "PROJECT",
                 project_name: str = "Construction Project",
                 project_short_name: Optional[str] = None,
                 planned_start: Optional[date] = None,
                 hours_per_day: float = 8.0,
                 calendar: Optional[WorkCalendar] = None,
                 currency: str = "USD"):
        self.project_id = project_id
        self.project_name = project_name
        self.project_short_name = (project_short_name or project_id)[:20]
        self.planned_start = planned_start or date.today()
        self.hours_per_day = float(hours_per_day)
        self.calendar = calendar or WorkCalendar(hours_per_day=self.hours_per_day)
        self.currency = currency
        self.default_calendar_name = "Standard 5 day workweek"
        self.activities: "OrderedDict[str, P6Activity]" = OrderedDict()
        self.relationships: List[P6Relationship] = []
        self.resources: "OrderedDict[str, P6Resource]" = OrderedDict()
        self.assignments: List[P6Assignment] = []
        self.wbs_names: Dict[str, str] = {}
        self.wbs_separator: Optional[str] = None
        self.import_warnings: List[str] = []

    # -- building ----------------------------------------------------------
    def add_activity(self, activity_id: str, name: str,
                     duration_days: float = 0.0,
                     activity_type: "Union[str, ActivityType]" = ActivityType.TASK_DEPENDENT,
                     wbs: str = "Project",
                     start: Optional[date] = None,
                     finish: Optional[date] = None,
                     percent_complete: float = 0.0,
                     actual_start: Optional[date] = None,
                     actual_finish: Optional[date] = None,
                     constraint_type: Optional[str] = None,
                     constraint_date: Optional[date] = None,
                     notes: str = "") -> P6Activity:
        act = P6Activity(
            activity_id=str(activity_id), name=str(name),
            duration_days=0.0 if ActivityType.parse(activity_type).is_milestone else float(duration_days or 0.0),
            activity_type=ActivityType.parse(activity_type), wbs=str(wbs or "Project"),
            start=start, finish=finish, actual_start=actual_start, actual_finish=actual_finish,
            percent_complete=float(percent_complete or 0.0),
            constraint_type=constraint_type, constraint_date=constraint_date, notes=notes or "",
        )
        self.activities[act.activity_id] = act
        return act

    def add_milestone(self, activity_id: str, name: str, finish: bool = False,
                      wbs: str = "Project", **kwargs: Any) -> P6Activity:
        at = ActivityType.FINISH_MILESTONE if finish else ActivityType.START_MILESTONE
        return self.add_activity(activity_id, name, 0, at, wbs=wbs, **kwargs)

    def add_relationship(self, predecessor: str, successor: str,
                         dep_type: "Union[str, DependencyType]" = DependencyType.FS,
                         lag_days: float = 0.0, comments: str = "") -> P6Relationship:
        rel = P6Relationship(str(predecessor), str(successor),
                             DependencyType.parse(dep_type), float(lag_days or 0.0),
                             comments or "")
        self.relationships.append(rel)
        return rel

    def add_resource(self, resource_id: str, name: str,
                     resource_type: str = "Labor", unit: str = "h") -> P6Resource:
        res = P6Resource(str(resource_id), str(name), resource_type, unit)
        self.resources[res.resource_id] = res
        return res

    def assign_resource(self, activity_id: str, resource_id: str,
                        planned_units: float = 0.0,
                        cost_per_unit: float = 0.0) -> P6Assignment:
        asn = P6Assignment(str(activity_id), str(resource_id),
                           float(planned_units or 0.0), float(cost_per_unit or 0.0))
        self.assignments.append(asn)
        return asn

    # -- import ------------------------------------------------------------
    @staticmethod
    def _as_date(value: Any) -> Optional[date]:
        if value is None:
            return None
        try:
            if pd.isna(value):
                return None
        except (TypeError, ValueError):
            pass
        if isinstance(value, datetime):
            return value.date()
        if isinstance(value, date) and not isinstance(value, datetime):
            return value
        ts = pd.to_datetime(value, errors="coerce")
        if pd.isna(ts):
            return None
        return ts.date()

    @staticmethod
    def _as_float(value: Any, default: float = 0.0) -> float:
        if value is None:
            return default
        try:
            if pd.isna(value):
                return default
        except (TypeError, ValueError):
            pass
        try:
            return float(value)
        except (TypeError, ValueError):
            return default

    @classmethod
    def _resolve_columns(cls, df: pd.DataFrame) -> Dict[str, str]:
        lookup = {str(c).strip().lower(): c for c in df.columns}
        resolved: Dict[str, str] = {}
        for field_name, aliases in cls.COLUMN_ALIASES.items():
            for alias in aliases:
                if alias in lookup:
                    resolved[field_name] = lookup[alias]
                    break
        return resolved

    @staticmethod
    def parse_predecessor_token(token: str,
                                known_ids: Iterable[str]) -> Optional[Tuple[str, DependencyType, float]]:
        raw = str(token).strip()
        if not raw:
            return None
        if ":" in raw:
            m = _PRED_COLON_RE.match(raw)
            if m and m.group("id"):
                dep_type = DependencyType.parse(m.group("type") or "FS")
                lag = float(m.group("lag") or 0.0)
                return m.group("id").strip(), dep_type, lag
        m = _PRED_SPACE_RE.match(raw)
        if m:
            return (m.group("id").strip(), DependencyType.parse(m.group("type")),
                    float(m.group("lag") or 0.0))
        # "ID+3" / "ID-2": resolve against the longest known id so activity
        # codes that themselves contain hyphens (e.g. "A-100") are not split.
        known = set(known_ids)
        best: Optional[Tuple[str, float]] = None
        for candidate in known:
            if raw.startswith(candidate):
                rest = raw[len(candidate):].strip()
                if re.fullmatch(r"[+-]\d+(?:\.\d+)?", rest):
                    if best is None or len(candidate) > len(best[0]):
                        best = (candidate, float(rest))
        if best:
            return best[0], DependencyType.FS, best[1]
        return raw, DependencyType.FS, 0.0

    @classmethod
    def from_dataframe(cls, df: pd.DataFrame,
                       project_id: str = "PROJECT",
                       project_name: str = "Construction Project",
                       planned_start: Optional[date] = None,
                       hours_per_day: float = 8.0,
                       wbs_separator: Optional[str] = None,
                       default_wbs: str = "Project",
                       **column_overrides: str) -> "P6Schedule":
        """Build a schedule from an activity table.

        Recognised columns are listed in ``COLUMN_ALIASES``; pass e.g.
        ``activity_id="Task Code"`` to override detection.
        """
        schedule = cls(project_id=project_id, project_name=project_name,
                       planned_start=planned_start, hours_per_day=hours_per_day)
        columns = cls._resolve_columns(df)
        columns.update({k: v for k, v in column_overrides.items() if v})

        if "activity_id" not in columns:
            raise ValueError("Could not find an activity id column. "
                             "Pass activity_id='<column name>' explicitly.")
        if "name" not in columns:
            raise ValueError("Could not find an activity name column. "
                             "Pass name='<column name>' explicitly.")

        schedule.wbs_separator = wbs_separator

        records: List[Dict[str, Any]] = []
        seen = set()
        for _, row in df.iterrows():
            aid = str(row[columns["activity_id"]]).strip()
            if not aid or aid.lower() in ("nan", "none"):
                continue
            if aid in seen:
                schedule.import_warnings.append(f"Duplicate activity id '{aid}' ignored.")
                continue
            seen.add(aid)

            type_value = (row[columns["activity_type"]]
                          if "activity_type" in columns else None)
            if type_value is None or (not isinstance(type_value, str) and pd.isna(type_value)):
                activity_type = ActivityType.TASK_DEPENDENT
            else:
                activity_type = ActivityType.parse(type_value)
            name_value = row[columns["name"]] if "name" in columns else aid
            if name_value is None or (not isinstance(name_value, str) and pd.isna(name_value)):
                name_value = aid
            records.append({
                "activity_id": aid,
                "name": str(name_value).strip() or aid,
                "duration": cls._as_float(row[columns["duration"]]) if "duration" in columns else 0.0,
                "wbs": (str(row[columns["wbs"]]).strip()
                        if "wbs" in columns and not pd.isna(row[columns["wbs"]]) else default_wbs),
                "start": cls._as_date(row[columns["start"]]) if "start" in columns else None,
                "finish": cls._as_date(row[columns["finish"]]) if "finish" in columns else None,
                "percent_complete": (cls._as_float(row[columns["percent_complete"]])
                                     if "percent_complete" in columns else 0.0),
                "actual_start": cls._as_date(row[columns["actual_start"]]) if "actual_start" in columns else None,
                "actual_finish": cls._as_date(row[columns["actual_finish"]]) if "actual_finish" in columns else None,
                "constraint_type": (str(row[columns["constraint_type"]]).strip()
                                    if "constraint_type" in columns and not pd.isna(row[columns["constraint_type"]])
                                    else None),
                "constraint_date": (cls._as_date(row[columns["constraint_date"]])
                                    if "constraint_date" in columns else None),
                "activity_type": activity_type,
                "resources": (str(row[columns["resources"]])
                              if "resources" in columns and not pd.isna(row[columns["resources"]]) else ""),
                "predecessors": (str(row[columns["predecessors"]])
                                 if "predecessors" in columns and not pd.isna(row[columns["predecessors"]]) else ""),
            })

        for rec in records:
            schedule.add_activity(
                rec["activity_id"], rec["name"], rec["duration"], rec["activity_type"],
                wbs=rec["wbs"], start=rec["start"], finish=rec["finish"],
                percent_complete=rec["percent_complete"],
                actual_start=rec["actual_start"], actual_finish=rec["actual_finish"],
                constraint_type=rec["constraint_type"], constraint_date=rec["constraint_date"])

        if schedule.wbs_separator is None:
            schedule.wbs_separator = schedule._detect_wbs_separator(
                [r["wbs"] for r in records])

        known_ids = list(schedule.activities.keys())
        for rec in records:
            token_blob = rec["predecessors"]
            if not token_blob:
                continue
            for token in re.split(r"[,;]", token_blob):
                parsed = cls.parse_predecessor_token(token, known_ids)
                if parsed:
                    pred, dep_type, lag = parsed
                    schedule.add_relationship(pred, rec["activity_id"], dep_type, lag)

            if rec["resources"]:
                for name in re.split(r"[,;]", rec["resources"]):
                    name = name.strip()
                    if not name:
                        continue
                    if name not in schedule.resources:
                        schedule.add_resource(name, name)
                    units = max(0.0, rec["duration"]) * schedule.hours_per_day
                    schedule.assign_resource(rec["activity_id"], name, planned_units=units)

        return schedule

    @classmethod
    def from_excel(cls, path: Union[str, Path], sheet_name: Any = 0,
                   **kwargs: Any) -> "P6Schedule":
        df = pd.read_excel(path, sheet_name=sheet_name)
        return cls.from_dataframe(df, **kwargs)

    @classmethod
    def from_csv(cls, path: Union[str, Path], **kwargs: Any) -> "P6Schedule":
        df = pd.read_csv(path)
        return cls.from_dataframe(df, **kwargs)

    def _detect_wbs_separator(self, paths: Iterable[str]) -> str:
        joined = " ".join(str(p) for p in paths)
        for sep in ("::", ">", "/", "\\"):
            if sep in joined:
                return sep
        if "." in joined:
            return "."
        return "."

    # -- logic / dates -----------------------------------------------------
    def _adjacency(self):
        known = set(self.activities)
        adj = defaultdict(list)
        incoming = defaultdict(list)
        for rel in self.relationships:
            if rel.predecessor in known and rel.successor in known:
                adj[rel.predecessor].append(rel.successor)
                incoming[rel.successor].append(rel)
        return adj, incoming

    def topological_order(self) -> Optional[List[str]]:
        adj, _ = self._adjacency()
        indegree = {aid: 0 for aid in self.activities}
        for succs in adj.values():
            for s in succs:
                indegree[s] += 1
        queue = deque([a for a, d in indegree.items() if d == 0])
        order: List[str] = []
        while queue:
            node = queue.popleft()
            order.append(node)
            for nxt in adj[node]:
                indegree[nxt] -= 1
                if indegree[nxt] == 0:
                    queue.append(nxt)
        return order if len(order) == len(self.activities) else None

    def compute_dates(self) -> "P6Schedule":
        """Forward-pass start/finish in working days, honouring FS/SS/FF/SF + lag."""
        order = self.topological_order()
        if order is None:
            raise ValueError("Cycle detected in the schedule network. Run validate() for details.")
        cal = self.calendar

        for aid in order:
            act = self.activities[aid]
            duration = int(round(act.duration_days)) if not act.is_milestone else 0
            candidate = act.start or self.planned_start

            for rel in self._incoming_map().get(aid, []):
                pred = self.activities.get(rel.predecessor)
                if pred is None:
                    continue
                p_start = pred.planned_start() or self.planned_start
                p_finish = pred.planned_finish() or p_start
                lag = int(round(rel.lag_days))
                if rel.dep_type == DependencyType.FS:
                    cand = cal.add_working_days(p_finish, 1 + lag)
                elif rel.dep_type == DependencyType.SS:
                    cand = cal.add_working_days(p_start, lag)
                elif rel.dep_type == DependencyType.FF:
                    target_finish = cal.add_working_days(p_finish, lag)
                    cand = cal.subtract_working_days(target_finish, max(0, duration - 1))
                else:  # SF
                    target_finish = cal.add_working_days(p_start, lag)
                    cand = cal.subtract_working_days(target_finish, max(0, duration - 1))
                if candidate is None or cand > candidate:
                    candidate = cand

            candidate = cal.snap_forward(candidate)
            act.computed_start = candidate
            if act.is_milestone or duration <= 0:
                act.computed_finish = candidate
            else:
                # a D-day activity that starts on day 1 finishes on day D
                act.computed_finish = cal.add_working_days(candidate, duration - 1)
        return self

    def _incoming_map(self):
        _, incoming = self._adjacency()
        return incoming

    def project_finish(self) -> Optional[date]:
        finishes = [a.planned_finish() for a in self.activities.values() if a.planned_finish()]
        return max(finishes) if finishes else None

    def project_start(self) -> Optional[date]:
        starts = [a.planned_start() for a in self.activities.values() if a.planned_start()]
        return min(starts) if starts else None

    # -- validation --------------------------------------------------------
    def validate(self) -> List[ValidationIssue]:
        issues: List[ValidationIssue] = []
        if not self.activities:
            issues.append(ValidationIssue("error", "NO_ACTIVITIES",
                                          "Schedule contains no activities."))
        for aid, act in self.activities.items():
            if not act.name or not str(act.name).strip():
                issues.append(ValidationIssue("warning", "MISSING_NAME",
                                              "Activity has no name.", aid))
            if act.is_milestone and act.duration_days not in (0, 0.0):
                issues.append(ValidationIssue("error", "MILESTONE_DURATION",
                                              "Milestone must have zero duration.", aid))
            if not act.is_milestone and act.duration_days < 0:
                issues.append(ValidationIssue("error", "NEGATIVE_DURATION",
                                              "Duration cannot be negative.", aid))
            if not 0 <= act.percent_complete <= 100:
                issues.append(ValidationIssue("error", "BAD_PERCENT_COMPLETE",
                                              "Percent complete must be 0-100.", aid))
            if act.actual_start and act.actual_finish and act.actual_finish < act.actual_start:
                issues.append(ValidationIssue("error", "ACTUAL_DATES",
                                              "Actual finish is before actual start.", aid))
            if act.actual_finish and not act.actual_start:
                issues.append(ValidationIssue("warning", "ACTUAL_FINISH_WITHOUT_START",
                                              "Actual finish set without actual start.", aid))
            if act.start and act.computed_start and act.start < act.computed_start:
                issues.append(ValidationIssue("warning", "DATE_CONSTRAINT_CONFLICT",
                                              "Explicit start conflicts with logic; logic wins.", aid))

        known = set(self.activities)
        seen_pairs = set()
        for rel in self.relationships:
            if rel.predecessor not in known:
                issues.append(ValidationIssue("error", "DANGLING_PREDECESSOR",
                                              f"Predecessor '{rel.predecessor}' not found.",
                                              rel.successor))
            if rel.successor not in known:
                issues.append(ValidationIssue("error", "DANGLING_SUCCESSOR",
                                              f"Successor '{rel.successor}' not found.",
                                              rel.predecessor))
            if rel.predecessor == rel.successor:
                issues.append(ValidationIssue("error", "SELF_REFERENCE",
                                              "Activity cannot precede itself.", rel.predecessor))
            key = (rel.predecessor, rel.successor, rel.dep_type.value)
            if key in seen_pairs:
                issues.append(ValidationIssue("warning", "DUPLICATE_RELATIONSHIP",
                                              "Duplicate relationship.", rel.successor))
            seen_pairs.add(key)

        if self.activities and self.topological_order() is None:
            issues.append(ValidationIssue("error", "CYCLE",
                                          "The relationship network contains a cycle."))

        known_res = set(self.resources)
        for asn in self.assignments:
            if asn.activity_id not in known:
                issues.append(ValidationIssue("error", "ASSIGNMENT_ACTIVITY",
                                              "Resource assigned to unknown activity.",
                                              asn.activity_id))
            if asn.resource_id not in known_res:
                issues.append(ValidationIssue("error", "ASSIGNMENT_RESOURCE",
                                              f"Unknown resource '{asn.resource_id}'.", asn.activity_id))
        return issues

    def has_errors(self) -> bool:
        return any(i.severity == "error" for i in self.validate())

    # -- WBS ---------------------------------------------------------------
    def wbs_nodes(self) -> List[Dict[str, Any]]:
        """Return ordered WBS nodes with cumulative codes and parents."""
        sep = self.wbs_separator or "."
        root_code = self.project_short_name or "PROJECT"
        nodes: "OrderedDict[str, Dict[str, Any]]" = OrderedDict()
        nodes[root_code] = {"code": root_code, "name": self.project_name, "parent": None}

        for act in self.activities.values():
            raw = (act.wbs or "").strip()
            if not raw or raw.lower() in ("project", "root"):
                continue
            parts = [p.strip() for p in raw.split(sep) if p.strip()]
            parent = root_code
            cumulative: List[str] = []
            for part in parts:
                cumulative.append(part)
                code = sep.join(cumulative)
                if code not in nodes:
                    nodes[code] = {
                        "code": code,
                        "name": self.wbs_names.get(code, part),
                        "parent": parent,
                    }
                parent = code
        return list(nodes.values())

    def wbs_code_for(self, activity: P6Activity) -> str:
        raw = (activity.wbs or "").strip()
        root_code = self.project_short_name or "PROJECT"
        if not raw or raw.lower() in ("project", "root"):
            return root_code
        return raw

    # -- exports -----------------------------------------------------------
    def to_dataframe(self) -> pd.DataFrame:
        rows = []
        preds_by_activity = defaultdict(list)
        for rel in self.relationships:
            sign = "+" if rel.lag_days >= 0 else ""
            preds_by_activity[rel.successor].append(
                f"{rel.predecessor}:{rel.dep_type.value}{sign}{rel.lag_days:g}")
        for act in self.activities.values():
            rows.append({
                "Activity ID": act.activity_id,
                "Activity Name": act.name,
                "WBS": self.wbs_code_for(act),
                "Duration (days)": act.duration_days,
                "Activity Type": act.activity_type.value,
                "Start": act.planned_start(),
                "Finish": act.planned_finish(),
                "% Complete": act.percent_complete,
                "Predecessors": ", ".join(preds_by_activity.get(act.activity_id, [])),
            })
        return pd.DataFrame(rows)

    def to_excel(self, output_path: Union[str, Path]) -> str:
        self.compute_dates()
        schedule_df = self.to_dataframe()
        rel_df = pd.DataFrame([{
            "Predecessor": r.predecessor,
            "Successor": r.successor,
            "Type": r.dep_type.value,
            "Lag (days)": r.lag_days,
        } for r in self.relationships])
        wbs_df = pd.DataFrame(self.wbs_nodes())
        resource_df = pd.DataFrame([{
            "Resource ID": r.resource_id, "Name": r.name, "Type": r.resource_type,
        } for r in self.resources.values()])
        with pd.ExcelWriter(output_path, engine="openpyxl") as writer:
            schedule_df.to_excel(writer, sheet_name="Activities", index=False)
            rel_df.to_excel(writer, sheet_name="Relationships", index=False)
            wbs_df.to_excel(writer, sheet_name="WBS", index=False)
            resource_df.to_excel(writer, sheet_name="Resources", index=False)
        return str(output_path)

    def summary(self) -> Dict[str, Any]:
        issues = self.validate()
        return {
            "project_id": self.project_id,
            "project_name": self.project_name,
            "activities": len(self.activities),
            "milestones": sum(1 for a in self.activities.values() if a.is_milestone),
            "relationships": len(self.relationships),
            "resources": len(self.resources),
            "planned_start": self.planned_start,
            "project_finish": self.project_finish(),
            "errors": sum(1 for i in issues if i.severity == "error"),
            "warnings": sum(1 for i in issues if i.severity == "warning"),
        }
```

### 2. P6 XML exporter

```python
# ---------------------------------------------------------------------------
# P6 XML exporter
# ---------------------------------------------------------------------------
class P6XMLExporter:
    """Serialize a P6Schedule to Primavera P6 XML (APIBusinessObjects)."""

    #: Namespace is version-specific; match your target P6 version.
    NAMESPACE = "http://xmlns.oracle.com/Primavera/P6/V19.12/API/BusinessObjects"

    def __init__(self, schedule: P6Schedule, namespace: Optional[str] = None,
                 hour_of_day: int = 8):
        self.schedule = schedule
        self.namespace = namespace or self.NAMESPACE
        self.hour_of_day = hour_of_day

    # -- helpers -----------------------------------------------------------
    def _q(self, tag: str) -> str:
        return f"{{{self.namespace}}}{tag}"

    def _sub(self, parent: ET.Element, tag: str, text: Any = None,
             attrib: Optional[Dict[str, str]] = None) -> ET.Element:
        el = ET.SubElement(parent, self._q(tag), attrib or {})
        if text is not None:
            el.text = str(text)
        return el

    def _dt(self, d: Optional[date]) -> Optional[str]:
        if d is None:
            return None
        if isinstance(d, datetime):
            return d.strftime("%Y-%m-%dT%H:%M:%S")
        return datetime(d.year, d.month, d.day, self.hour_of_day, 0, 0).strftime("%Y-%m-%dT%H:%M:%S")

    def _status(self, act: P6Activity) -> str:
        if act.actual_finish or act.percent_complete >= 100:
            return "Completed"
        if act.actual_start or act.percent_complete > 0:
            return "In Progress"
        return "Not Started"

    # -- build -------------------------------------------------------------
    def build(self) -> ET.Element:
        sched = self.schedule
        sched.compute_dates()

        ET.register_namespace("", self.namespace)
        ET.register_namespace("xsi", XSI_NS)
        root = ET.Element(self._q("APIBusinessObjects"), {
            f"{{{XSI_NS}}}schemaLocation": f"{self.namespace} {self.namespace}.xsd"
        })

        proj_id = 1
        cal_id = 1
        wbs_nodes = sched.wbs_nodes()
        wbs_ids = {node["code"]: 100 + i for i, node in enumerate(wbs_nodes)}
        activity_ids = {aid: 10000 + i for i, aid in enumerate(sched.activities)}
        resource_ids = {rid: 20000 + i for i, rid in enumerate(sched.resources)}
        assignment_ids = list(range(30000, 30000 + len(sched.assignments)))
        relationship_ids = list(range(40000, 40000 + len(sched.relationships)))

        # Project
        proj_el = ET.SubElement(root, self._q("Project"))
        for tag, value in [
            ("ObjectId", proj_id), ("Id", sched.project_id), ("Name", sched.project_name),
            ("Type", "Project"), ("PlannedStartDate", self._dt(sched.planned_start)),
            ("DataDate", self._dt(sched.planned_start)),
            ("DefaultCalendarObjectId", cal_id),
            ("ActivityDefaultCalendarObjectId", cal_id),
            ("HoursPerDay", sched.hours_per_day),
            ("Status", "Active"),
            ("WBSObjectId", wbs_ids[wbs_nodes[0]["code"]]),
            ("StartDate", self._dt(sched.project_start())),
            ("FinishDate", self._dt(sched.project_finish())),
            ("PlannedFinishDate", self._dt(sched.project_finish())),
        ]:
            if value is not None:
                self._sub(proj_el, tag, value)

        # Calendar
        cal_el = ET.SubElement(root, self._q("Calendar"))
        for tag, value in [
            ("ObjectId", cal_id), ("Name", sched.default_calendar_name),
            ("Type", "Global"), ("HoursPerDay", sched.hours_per_day),
            ("HoursPerWeek", sched.hours_per_day * 5),
            ("HoursPerMonth", sched.hours_per_day * 21.5),
            ("HoursPerYear", sched.hours_per_day * 260),
            ("DefaultFlag", "Y"), ("IsDefault", "true"),
        ]:
            self._sub(cal_el, tag, value)
        work_week = ET.SubElement(cal_el, self._q("StandardWorkWeek"))
        for day_name in ("Monday", "Tuesday", "Wednesday", "Thursday", "Friday"):
            day_el = ET.SubElement(work_week, self._q(day_name))
            for start, finish in (("08:00", "12:00"), ("13:00", "17:00")):
                wt = ET.SubElement(day_el, self._q("WorkTime"))
                self._sub(wt, "Start", start)
                self._sub(wt, "Finish", finish)
        for day_name in ("Saturday", "Sunday"):
            ET.SubElement(work_week, self._q(day_name))

        # WBS
        sep = sched.wbs_separator or "."
        for node in wbs_nodes:
            wbs_el = ET.SubElement(root, self._q("WBS"))
            self._sub(wbs_el, "ObjectId", wbs_ids[node["code"]])
            self._sub(wbs_el, "Id", node["code"])
            self._sub(wbs_el, "Name", node["name"])
            self._sub(wbs_el, "ProjectObjectId", proj_id)
            if node["parent"]:
                self._sub(wbs_el, "ParentObjectId", wbs_ids[node["parent"]])
            self._sub(wbs_el, "SequenceNumber", wbs_nodes.index(node) + 1)
            self._sub(wbs_el, "Code", node["code"])

        # Activities
        for act in sched.activities.values():
            act_el = ET.SubElement(root, self._q("Activity"))
            wbs_code = sched.wbs_code_for(act)
            duration_hours = act.duration_days * sched.hours_per_day
            fields = [
                ("ObjectId", activity_ids[act.activity_id]),
                ("Id", act.activity_id),
                ("Name", act.name),
                ("ProjectObjectId", proj_id),
                ("WBSObjectId", wbs_ids.get(wbs_code, wbs_ids[wbs_nodes[0]["code"]])),
                ("CalendarObjectId", cal_id),
                ("ActivityType", act.activity_type.value),
                ("DurationType", "Fixed Duration & Units"),
                ("PercentCompleteType", "Duration"),
                ("PlannedDuration", duration_hours),
                ("RemainingDuration", duration_hours),
                ("AtCompletionDuration", duration_hours),
                ("PlannedStartDate", self._dt(act.planned_start())),
                ("PlannedFinishDate", self._dt(act.planned_finish())),
                ("StartDate", self._dt(act.planned_start())),
                ("FinishDate", self._dt(act.planned_finish())),
                ("ActualStartDate", self._dt(act.actual_start)),
                ("ActualFinishDate", self._dt(act.actual_finish)),
                ("PercentComplete", act.percent_complete),
                ("Status", self._status(act)),
            ]
            if act.constraint_type:
                fields += [("PrimaryConstraintType", act.constraint_type),
                           ("PrimaryConstraintDate", self._dt(act.constraint_date))]
            if act.notes:
                fields.append(("Notes", act.notes))
            for tag, value in fields:
                if value is not None:
                    self._sub(act_el, tag, value)

        # Resources
        for res in sched.resources.values():
            res_el = ET.SubElement(root, self._q("Resource"))
            for tag, value in [
                ("ObjectId", resource_ids[res.resource_id]),
                ("Id", res.resource_id), ("Name", res.name),
                ("ResourceType", res.resource_type),
                ("UnitOfMeasure", res.unit), ("IsActive", "true"),
            ]:
                self._sub(res_el, tag, value)

        # Resource assignments
        for asn, obj_id in zip(sched.assignments, assignment_ids):
            asn_el = ET.SubElement(root, self._q("ResourceAssignment"))
            for tag, value in [
                ("ObjectId", obj_id), ("ProjectObjectId", proj_id),
                ("ActivityObjectId", activity_ids.get(asn.activity_id, 0)),
                ("ResourceObjectId", resource_ids.get(asn.resource_id, 0)),
                ("PlannedUnits", asn.planned_units),
                ("RemainingUnits", asn.planned_units),
                ("PlannedCost", asn.planned_cost),
                ("RemainingCost", asn.planned_cost),
            ]:
                self._sub(asn_el, tag, value)

        # Relationships
        for rel, obj_id in zip(sched.relationships, relationship_ids):
            rel_el = ET.SubElement(root, self._q("Relationship"))
            for tag, value in [
                ("ObjectId", obj_id),
                ("PredecessorProjectObjectId", proj_id),
                ("SuccessorProjectObjectId", proj_id),
                ("PredecessorActivityObjectId", activity_ids.get(rel.predecessor, 0)),
                ("SuccessorActivityObjectId", activity_ids.get(rel.successor, 0)),
                ("Type", rel.dep_type.p6_xml_label),
                ("Lag", rel.lag_days * sched.hours_per_day),
            ]:
                self._sub(rel_el, tag, value)
            if rel.comments:
                self._sub(rel_el, "Comments", rel.comments)

        if hasattr(ET, "indent"):
            ET.indent(root, space="  ")
        return root

    def to_string(self) -> str:
        root = self.build()
        return ET.tostring(root, encoding="unicode", xml_declaration=False)

    def export(self, output_path: Union[str, Path]) -> str:
        root = self.build()
        tree = ET.ElementTree(root)
        tree.write(output_path, encoding="utf-8", xml_declaration=True)
        return str(output_path)
```

### 3. XER exporter

```python
# ---------------------------------------------------------------------------
# XER exporter
# ---------------------------------------------------------------------------
class XERExporter:
    """Serialize a P6Schedule to a tab-delimited Primavera XER file."""

    #: P6 constraint-type codes (matches the XER enum). There is deliberately no
    #: "CS_None": an activity with no constraint carries an empty cstr_type.
    CONSTRAINT_CODES = {
        "CS_ALAP", "CS_MEO", "CS_MEOA", "CS_MEOB", "CS_MANDFIN",
        "CS_MANDSTART", "CS_MSO", "CS_MSOA", "CS_MSOB",
    }
    #: Friendly aliases, so callers may pass "MSO" instead of "CS_MSO".
    CONSTRAINT_ALIASES = {
        "ALAP": "CS_ALAP", "MEO": "CS_MEO", "MEOA": "CS_MEOA", "MEOB": "CS_MEOB",
        "MANDFIN": "CS_MANDFIN", "MANDSTART": "CS_MANDSTART", "MSO": "CS_MSO",
        "MSOA": "CS_MSOA", "MSOB": "CS_MSOB",
    }

    def __init__(self, schedule: P6Schedule, xer_version: str = "19.12",
                 export_user: str = "admin", company: str = "DDC",
                 database: str = "dbxDatabaseNoName",
                 calendar_data: Optional[str] = None):
        self.schedule = schedule
        self.xer_version = xer_version
        self.export_user = export_user
        self.company = company
        self.database = database
        #: P6 encodes the working week in this blob. A valid one is built from the
        #: schedule calendar; pass calendar_data to override it with a known-good
        #: value copied from a real P6 export of your target version.
        self.calendar_data = calendar_data

    def _dt(self, d: Optional[date], end: bool = False, midnight: bool = False) -> str:
        """P6 stamps dates as ``YYYY-MM-DD HH:MM``.

        Starts default to 08:00 and finishes to 08:00 + ``hours_per_day`` (16:00
        for an 8-hour day); project-level plan dates use midnight.
        """
        if d is None:
            return ""
        if midnight:
            hour = 0
        elif end:
            hour = int(8 + self.schedule.hours_per_day) % 24
        else:
            hour = 8
        return datetime(d.year, d.month, d.day, hour, 0, 0).strftime("%Y-%m-%d %H:%M")

    def _constraint_code(self, value: Optional[str]) -> str:
        """Normalise a constraint to a real P6 code, or "" for none/invalid."""
        if not value:
            return ""
        token = str(value).strip().upper().replace(" ", "").replace("_", "")
        if token in self.CONSTRAINT_CODES:
            return token
        return self.CONSTRAINT_ALIASES.get(token, "")

    @staticmethod
    def _excel_serial(d: date) -> int:
        """Excel/Lotus serial date (days since 1899-12-30), used by clndr_data."""
        return (d - date(1899, 12, 30)).days

    def _clndr_data(self) -> str:
        """Encode the schedule calendar as P6's opaque ``clndr_data`` string.

        Grammar verified against real P6 exports: day numbers run 1=Sunday ..
        7=Saturday, working days hold a single 08:00 shift, and holidays are
        encoded as Excel-serial-date exceptions.
        """
        if self.calendar_data is not None:
            return self.calendar_data
        cal = self.schedule.calendar
        end_hour = int(8 + self.schedule.hours_per_day)
        end = f"{end_hour:02d}:00"
        days = []
        for p6_day in range(1, 8):
            # p6_day: 1=Sun..7=Sat  ->  Python weekday: Mon=0..Sun=6
            python_weekday = (p6_day - 2) % 7
            if python_weekday in cal.workdays:
                days.append(f"(0||{p6_day}()((0||0(s|08:00|f|{end})())))")
            else:
                days.append(f"(0||{p6_day}()())")
        exceptions = "".join(
            f"(0||{i}(d|{self._excel_serial(h)})())"
            for i, h in enumerate(sorted(cal.holidays))
        )
        return (
            "(0||CalendarData()((0||DaysOfWeek()(" + "".join(days) + "))"
            "(0||VIEW(ShowTotal|Y)())(0||Exceptions()(" + exceptions + "))))"
        )

    def _status(self, act: P6Activity) -> str:
        if act.actual_finish or act.percent_complete >= 100:
            return "TK_Complete"
        if act.actual_start or act.percent_complete > 0:
            return "TK_Active"
        return "TK_NotStart"

    @staticmethod
    def _num(value: float) -> str:
        return f"{value:g}"

    def _section(self, lines: List[str], table: str, fields: List[str],
                 rows: List[List[Any]]) -> None:
        lines.append("%T\t" + table)
        lines.append("%F\t" + "\t".join(fields))
        for row in rows:
            cells = []
            for value in row:
                if value is None:
                    cells.append("")
                elif isinstance(value, float):
                    cells.append(self._num(value))
                else:
                    cells.append(str(value))
            lines.append("%R\t" + "\t".join(cells))

    def _maps(self):
        sched = self.schedule
        wbs_nodes = sched.wbs_nodes()
        wbs_ids = {node["code"]: i + 1 for i, node in enumerate(wbs_nodes)}
        task_ids = {aid: i + 1 for i, aid in enumerate(sched.activities)}
        resource_ids = {rid: i + 1 for i, rid in enumerate(sched.resources)}
        return wbs_nodes, wbs_ids, task_ids, resource_ids

    def build(self) -> str:
        sched = self.schedule
        sched.compute_dates()
        wbs_nodes, wbs_ids, task_ids, resource_ids = self._maps()
        root_wbs_id = wbs_ids[wbs_nodes[0]["code"]]
        project_finish = sched.project_finish() or sched.planned_start
        today = date.today().strftime("%Y-%m-%d")

        lines: List[str] = []
        # ERMHDR carries 9 tab-separated columns. The maintained XER parsers read
        # the user at index 4 and the currency at index 7 - i.e. columns 6 and 9 -
        # which is what real P6 exports look like.
        header = ["ERMHDR", self.xer_version, today, "Project",
                  self.export_user, self.export_user, self.database,
                  self.company, sched.currency]
        lines.append("\t".join(str(f) for f in header))

        # PROJECT
        self._section(
            lines, "PROJECT",
            ["proj_id", "proj_short_name", "clndr_id", "last_recalc_date",
             "plan_start_date", "plan_end_date", "def_duration_type",
             "def_complete_pct_type", "export_flag"],
            [[1, sched.project_short_name, 1,
              self._dt(sched.planned_start, midnight=True),
              self._dt(sched.planned_start, midnight=True),
              self._dt(project_finish, midnight=True),
              "DT_FixedDUR2", "CP_Phys", "Y"]])

        # CALENDAR
        self._section(
            lines, "CALENDAR",
            ["clndr_id", "default_flag", "clndr_name", "proj_id", "base_clndr_id",
             "clndr_type", "day_hr_cnt", "week_hr_cnt", "month_hr_cnt",
             "year_hr_cnt", "rsrc_private", "clndr_data"],
            [[1, "Y", sched.default_calendar_name, 1, "", "CA_Base",
              sched.hours_per_day, sched.hours_per_day * 5,
              round(sched.hours_per_day * 21.5, 2), sched.hours_per_day * 250,
              "N", self._clndr_data()]])

        # PROJWBS
        wbs_parent_by_code = {n["code"]: n["parent"] for n in wbs_nodes}
        self._section(
            lines, "PROJWBS",
            ["wbs_id", "proj_id", "obs_id", "seq_num", "proj_node_flag",
             "status_code", "wbs_short_name", "wbs_name", "parent_wbs_id"],
            [[wbs_ids[node["code"]], 1, "",
              i + 1, "N" if node["parent"] else "Y", "WS_Open",
              node["code"], node["name"],
              wbs_ids.get(wbs_parent_by_code.get(node["code"]), "") if node["parent"] else ""]
             for i, node in enumerate(wbs_nodes)])

        # TASK
        task_fields = [
            "task_id", "proj_id", "wbs_id", "clndr_id", "task_code", "task_name",
            "task_type", "status_code", "target_drtn_hr_cnt", "remain_drtn_hr_cnt",
            "target_start_date", "target_end_date", "early_start_date", "early_end_date",
            "late_start_date", "late_end_date", "act_start_date", "act_end_date",
            "phys_complete_pct", "complete_pct_type", "duration_type",
            "cstr_type", "cstr_date", "free_float_hr_cnt", "total_float_hr_cnt",
        ]
        task_rows = []
        for act in sched.activities.values():
            wbs_code = sched.wbs_code_for(act)
            duration_hours = act.duration_days * sched.hours_per_day
            # P6 stores physical percent complete as 0-100 (not a 0-1 fraction).
            pct = max(0.0, min(100.0, float(act.percent_complete or 0.0)))
            remain_hours = (0.0 if act.activity_type.is_milestone
                            else duration_hours * (1 - pct / 100.0))
            task_rows.append([
                task_ids[act.activity_id], 1,
                wbs_ids.get(wbs_code, root_wbs_id), 1,
                act.activity_id, act.name,
                act.activity_type.xer_code, self._status(act),
                duration_hours, round(remain_hours, 4),
                self._dt(act.planned_start()), self._dt(act.planned_finish(), end=True),
                self._dt(act.planned_start()), self._dt(act.planned_finish(), end=True),
                "", "", self._dt(act.actual_start), self._dt(act.actual_finish, end=True),
                pct, "CP_Phys", "DT_FixedDUR2",
                self._constraint_code(act.constraint_type),
                self._dt(act.constraint_date), "", "",
            ])
        self._section(lines, "TASK", task_fields, task_rows)

        # Resources. The table is RSRC (upper case): "Rsrc" is not a valid XER table.
        if sched.resources:
            self._section(
                lines, "RSRC",
                ["rsrc_id", "parent_rsrc_id", "rsrc_seq_num", "rsrc_short_name",
                 "rsrc_name", "rsrc_type", "clndr_id", "def_qty_per_hr"],
                [[resource_ids[res.resource_id], 0, i + 1, res.resource_id,
                  res.name, res.xer_type, 1, 1]
                 for i, res in enumerate(sched.resources.values())])

        if sched.assignments:
            self._section(
                lines, "TASKRSRC",
                ["taskrsrc_id", "task_id", "proj_id", "rsrc_id", "target_qty",
                 "remain_qty", "act_reg_qty", "target_cost", "remain_cost",
                 "act_reg_cost", "cost_per_qty"],
                [[i + 1, task_ids.get(a.activity_id, ""), 1,
                  resource_ids.get(a.resource_id, ""), a.planned_units,
                  a.planned_units, 0, a.planned_cost, a.planned_cost, 0,
                  a.cost_per_unit] for i, a in enumerate(sched.assignments)])

        # TASKPRED
        self._section(
            lines, "TASKPRED",
            ["task_pred_id", "task_id", "pred_task_id", "proj_id", "pred_proj_id",
             "pred_type", "lag_hr_cnt", "comments", "float_path", "aref", "arls"],
            [[i + 1, task_ids.get(rel.successor, ""), task_ids.get(rel.predecessor, ""),
              1, 1, rel.dep_type.xer_code,
              rel.lag_days * sched.hours_per_day, rel.comments, "", "", ""]
             for i, rel in enumerate(sched.relationships)])

        lines.append("%E")
        return "\r\n".join(lines) + "\r\n"

    def export(self, output_path: Union[str, Path]) -> str:
        text = self.build()
        with open(output_path, "w", encoding="utf-8", newline="") as fh:
            fh.write(text)
        return str(output_path)
```

### 4. One-call API and command line

```python
# ---------------------------------------------------------------------------
# Convenience API + CLI
# ---------------------------------------------------------------------------
def export_schedule(df: pd.DataFrame, output_path: Union[str, Path],
                    fmt: str = "xml", **kwargs: Any) -> str:
    """One-call export: DataFrame -> P6 XML or XER."""
    schedule = P6Schedule.from_dataframe(df, **kwargs)
    errors = [i for i in schedule.validate() if i.severity == "error"]
    if errors:
        raise ValueError("Schedule validation failed:\n" + "\n".join(str(e) for e in errors))
    fmt = fmt.lower()
    if fmt == "xml":
        return P6XMLExporter(schedule).export(output_path)
    if fmt == "xer":
        return XERExporter(schedule).export(output_path)
    if fmt in ("xlsx", "excel"):
        return schedule.to_excel(output_path)
    raise ValueError("fmt must be 'xml', 'xer' or 'excel'")


def main(argv: Optional[List[str]] = None) -> int:
    import argparse
    parser = argparse.ArgumentParser(
        description="Export a CSV/Excel construction schedule to Primavera P6 XML/XER.")
    parser.add_argument("input", help="Activity table (.csv, .xlsx)")
    parser.add_argument("--sheet", default=0, help="Excel sheet name/index")
    parser.add_argument("--xml", dest="xml_out", help="Write P6 XML here")
    parser.add_argument("--xer", dest="xer_out", help="Write XER here")
    parser.add_argument("--excel", dest="excel_out", help="Write P6-friendly Excel here")
    parser.add_argument("--project-id", default="PROJECT")
    parser.add_argument("--project-name", default="Construction Project")
    parser.add_argument("--start", help="Project start date YYYY-MM-DD")
    parser.add_argument("--hours-per-day", type=float, default=8.0)
    parser.add_argument("--wbs-separator", default=None)
    args = parser.parse_args(argv)

    path = Path(args.input)
    df = pd.read_excel(path, sheet_name=args.sheet) if path.suffix.lower() in (".xlsx", ".xls") \
        else pd.read_csv(path)

    start = P6Schedule._as_date(args.start) if args.start else None
    schedule = P6Schedule.from_dataframe(
        df, project_id=args.project_id, project_name=args.project_name,
        planned_start=start, hours_per_day=args.hours_per_day,
        wbs_separator=args.wbs_separator)

    issues = schedule.validate()
    for issue in issues:
        print(issue)
    if any(i.severity == "error" for i in issues):
        print("Aborting: fix the errors above.")
        return 1

    for warning in schedule.import_warnings:
        print("WARNING:", warning)

    if args.xml_out:
        print("Wrote", P6XMLExporter(schedule).export(args.xml_out))
    if args.xer_out:
        print("Wrote", XERExporter(schedule).export(args.xer_out))
    if args.excel_out:
        print("Wrote", schedule.to_excel(args.excel_out))
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

## Quick Start

```python
import pandas as pd
from datetime import date
from p6_schedule_exporter import P6Schedule, P6XMLExporter, XERExporter

# 1. Build from the activity table your scheduling skill already produces
df = pd.read_excel("schedule.xlsx")          # columns: activity_id, name, duration,
                                             # wbs, predecessors, start, ...
schedule = P6Schedule.from_dataframe(
    df,
    project_id="TOWER-A",
    project_name="Tower A - Main Works",
    planned_start=date(2024, 6, 3),
    hours_per_day=8,
)

# 2. Validate before exporting (dangling logic, cycles, bad values)
issues = schedule.validate()
for issue in issues:
    print(issue)
assert not [i for i in issues if i.severity == "error"]

# 3. Forward-pass dates from FS/SS/FF/SF logic
schedule.compute_dates()
print(schedule.summary())

# 4. Write P6-loadable files
P6XMLExporter(schedule).export("tower_a.xml")
XERExporter(schedule).export("tower_a.xer")
schedule.to_excel("tower_a_p6_layout.xlsx")
```

### Command line

```bash
python p6_schedule_exporter.py schedule.xlsx \
    --project-id TOWER-A --project-name "Tower A" \
    --start 2024-06-03 \
    --xml tower_a.xml --xer tower_a.xer --excel tower_a.xlsx
```

## Common Use Cases

### 1. Excel/CSV -> P6 XML (most reliable import path)
```python
schedule = P6Schedule.from_excel("activities.xlsx", sheet_name="Schedule",
                                 project_name="Warehouse Ph.2",
                                 planned_start=date(2024, 9, 2))
errors = [i for i in schedule.validate() if i.severity == "error"]
if errors:
    raise SystemExit("\n".join(str(e) for e in errors))
P6XMLExporter(schedule).export("warehouse.xml")
```

### 2. Feed the critical-path-analyzer output straight in
```python
# critical-path-analyzer / cwicr-schedule-integrator return a DataFrame with
# activity_id, name, duration and predecessors - exactly what from_dataframe expects.
schedule = P6Schedule.from_dataframe(analyzer.get_schedule_dates(),
                                     planned_start=date(2024, 6, 1))
```

### 3. Cost-loaded export with resources
```python
schedule = P6Schedule.from_dataframe(df, planned_start=date(2024, 6, 3))
schedule.add_resource("LAB-01", "General Labour", "Labor", "h")
schedule.assign_resource("A-100", "LAB-01", planned_units=40, cost_per_unit=55.0)
P6XMLExporter(schedule).export("cost_loaded.xml")
```

### 4. XER for an older P6 release
```python
XERExporter(schedule, xer_version="16.1").export("legacy.xer")
```

### 5. One-call conversion with built-in validation
```python
from p6_schedule_exporter import export_schedule

export_schedule(df, "out.xml", fmt="xml",
                project_name="Metro Depot", planned_start=date(2024, 6, 3))
```

## Precedence notation accepted

The `predecessors` cell can contain one or more of the following, comma-separated:

| Token | Meaning |
|-------|---------|
| `A-100` | Finish-to-Start, zero lag |
| `A-100:FS+3` | Finish-to-Start, +3 working-day lag |
| `A-100:SS-2` | Start-to-Start, -2 day lag |
| `A-100:FF` | Finish-to-Finish |
| `A-100:SF+5` | Start-to-Finish, +5 day lag |
| `A-100 FS +3` | Type separated by spaces |
| `A-100+3` | FS with lag (resolved against the longest known activity code, so codes containing hyphens stay intact) |
| `C-330:FS,D-410:FS` | Multiple predecessors |

## Importing into Primavera P6

**P6 XML**
1. `File > Open` -> select the `.xml` file, or `Project > Import`.
2. Choose **Primavera P6 XML** as the format and map to a new/existing EPS.
3. Open the project and run **F9 (Schedule)** to let P6 recalculate with its own calendar.

**XER**
1. `File > Open` -> select the `.xer` file.
2. Pick the destination EPS and confirm.
3. Run **F9 (Schedule)**.

**Excel** - `File > Import` -> **Spreadsheet**, then map columns for Activities and Relationships.

## Format notes and caveats

- **Namespace/version** - P6 XML carries a version-specific namespace. Update
  `P6XMLExporter.NAMESPACE` to match your P6 version (e.g. `V8.2`, `V19.12`) if an older P6
  rejects the file. The `xsi:schemaLocation` is informational; P6 does not fetch it.
- **Duration and lag units** - P6 stores durations and relationship lags in **hours**. The
  exporter converts days using `hours_per_day` (default 8) for both XML `PlannedDuration`/`Lag`
  and XER `target_drtn_hr_cnt`/`lag_hr_cnt`.
- **Calendar** - XML emits a standard 5-day, 8-hour work week. XER `clndr_data` is P6's opaque,
  version-specific working-week blob; the exporter builds a valid one from the schedule calendar
  (day numbers 1=Sun..7=Sat, one 08:00 shift per working day, holidays as Excel-serial-date
  exceptions). Override it with `XERExporter(schedule, calendar_data="...")` if your P6 version
  needs an exact blob copied from one of its own exports.
- **ERMHDR** - the XER header is emitted as 9 tab-separated columns in the layout real P6 exports
  use (`ERMHDR <version> <date> Project <user> <user> <database> <company> <currency>`). The
  maintained XER parsers read the user at column 6 and the currency at column 9. P6 largely
  regenerates the header on import; the `%T`/`%F`/`%R` table structure is what matters.
- **XER column subset** - XER is self-describing: each `%T` block's `%F` row defines the columns
  for the `%R` rows that follow, so the exporter writes a minimal-but-valid subset of each table's
  real columns rather than all 60-70 version-specific fields. Every column it writes is a real P6
  column, and every enum value (`TT_*`, `DT_*`, `CP_*`, `CS_*`, `TK_*`, `RT_*`, `PR_*`) is a valid
  P6 code - checked against genuine P6 15.2/20.12 exports and Oracle's XER data map.
- **Resource unit** - `RSRC` deliberately omits a unit-of-measure column: its name differs between
  P6 versions (`unit_id` vs `unit_abbrev`) and an unknown column can break import. P6 supplies its
  own default; set units in P6 after import if you need them.
- **Percent complete** - stored the way P6 does, as `phys_complete_pct` in 0-100 with
  `complete_pct_type=CP_Phys`; remaining duration is derived from it. An activity with no
  constraint carries an empty `cstr_type` (P6 has no `CS_None` code).
- **Dates** - if an activity has an explicit `start`, it acts as an "early start no earlier than"
  constraint; precedence logic can push it later. Run P6's F9 recalc after import for authoritative
  dates, as your P6 calendar may differ from the built-in 5-day calendar.
- **Hours-per-day** - if the source schedule's `duration` is already in hours, set
  `hours_per_day=1`.
- **P6 trademark** - Primavera P6 is an Oracle product; this skill only creates importable files
  and does not require P6 to run.

## Validation checks

`P6Schedule.validate()` returns `severity` + `code` + `message` for each finding:

- `NO_ACTIVITIES`, `MISSING_NAME`, `MILESTONE_DURATION`, `NEGATIVE_DURATION`
- `BAD_PERCENT_COMPLETE`, `ACTUAL_DATES`, `ACTUAL_FINISH_WITHOUT_START`
- `DANGLING_PREDECESSOR`, `DANGLING_SUCCESSOR`, `SELF_REFERENCE`, `DUPLICATE_RELATIONSHIP`, `CYCLE`
- `ASSIGNMENT_ACTIVITY`, `ASSIGNMENT_RESOURCE`, `DATE_CONSTRAINT_CONFLICT`

Errors block a clean export; warnings do not. `export_schedule()` refuses to write when errors exist.

## Integration with other DDC skills

| Upstream skill | How it feeds this one |
|----------------|----------------------|
| `critical-path-analyzer` | Supplies `activity_id`, `name`, `duration`, `predecessors` |
| `cwicr-schedule-integrator` | Adds cost-loaded activities and resources |
| `bim-to-schedule-4d` | Supplies WBS/zone groupings |
| `oce-scheduling-4d5d` | Supplies FS/SS/FF task graph, milestones, critical path |
| `xml-reader` (`P6XMLReader`) | Reads P6 XML back for round-trip analysis |

## Resources
- **DDC Book**: Chapter 4.2 - Schedule Analysis
- **Oracle Primavera P6 XML / XER** import documentation for your installed version
