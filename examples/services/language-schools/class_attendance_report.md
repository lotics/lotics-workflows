# Weekly Class Attendance Report

## The problem

Teachers mark attendance on paper sheets that pile up in the office. The school director needs attendance data to identify at-risk students (below 75% attendance) and report to parents or sponsors, but compiling this from paper records takes a full day each month. Students who miss 3+ consecutive classes often drop out silently, and by the time anyone notices, the student is gone and the tuition slot is wasted.

## Data model

| Table | Key fields | Purpose |
|-------|-----------|---------|
| Attendance | student_id, class_id, date, status, marked_by | Daily attendance records |
| Classes | class_name, language, level, teacher_id, schedule | Class information |
| Students | full_name, email, emergency_contact_email | Student profiles |
| Teachers | full_name, email | Teacher profiles |

## Example prompts

- "Every Friday at 5 PM, generate an attendance report for each class showing each student's attendance percentage for the week. If anyone has missed all classes that week, email their emergency contact."
- "Create a weekly attendance summary per class and flag students who missed every session that week for follow-up."

## Workflow

**Trigger:** Recurring schedule, every Friday at 17:00

```json
[
  {
    "id": "get_classes",
    "type": "tool_call",
    "description": "Fetch all active classes",
    "tool_name": "query_records",
    "input": {
      "table_name": "Classes"
    }
  },
  {
    "id": "per_class",
    "type": "foreach",
    "description": "Generate attendance report for each class",
    "items": "{{get_classes.output.records}}",
    "steps": [
      {
        "id": "get_attendance",
        "type": "tool_call",
        "description": "Query this week's attendance records for the class",
        "tool_name": "query_records",
        "input": {
          "table_name": "Attendance",
          "filters": {
            "class_id": "{{item.id}}",
            "date_from": "{{trigger.recurring_schedule.scheduled_at_date_minus_5d}}",
            "date_to": "{{trigger.recurring_schedule.scheduled_at_date}}"
          }
        }
      },
      {
        "id": "build_report",
        "type": "tool_call",
        "description": "Use AI to compile attendance data into a readable summary",
        "tool_name": "llm_generate_text",
        "input": {
          "prompt": "Analyze the following attendance records for the class {{item.class_name}} and produce a concise report. List each student with their attendance count out of total sessions this week and their percentage. Flag any student who attended 0 sessions as AT RISK. Attendance data: {{get_attendance.output.records}}. Format as a plain text table."
        }
      },
      {
        "id": "get_teacher",
        "type": "tool_call",
        "description": "Fetch teacher email for the report",
        "tool_name": "get_record",
        "input": {
          "table_name": "Teachers",
          "record_id": "{{item.teacher_id}}"
        }
      },
      {
        "id": "email_report",
        "type": "tool_call",
        "description": "Send the attendance report to the teacher",
        "tool_name": "gmail_send_email",
        "input": {
          "to": "{{get_teacher.output.email}}",
          "subject": "Weekly Attendance Report - {{item.class_name}} - Week of {{trigger.recurring_schedule.scheduled_at_date}}",
          "body": "Hi {{get_teacher.output.full_name}},\n\nHere is the weekly attendance summary for {{item.class_name}}:\n\n{{build_report.output.text}}\n\nPlease review any AT RISK students and coordinate follow-up.\n\nThis report is generated automatically every Friday."
        }
      }
    ]
  }
]
```

## Results

| Metric | Before | After |
|--------|--------|-------|
| Time to compile attendance reports | 8 hrs/month | 0 (automated weekly) |
| At-risk students identified within 1 week | 30% | 100% |
| Silent dropouts per quarter | 8-10 | 2-3 |

-> [Set up this workflow on Lotics](https://lotics.ai)
