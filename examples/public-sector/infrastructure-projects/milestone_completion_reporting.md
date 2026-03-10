# Milestone Completion Reporting

## The problem

A state DOT manages capital improvement projects worth $5M-$200M each, with 10-40 milestones per project. Project managers submit milestone completions by updating a record and attaching inspection photos, daily logs, and engineer sign-off letters. The program office then has to manually compile a PDF progress report for the federal funding agency each time a milestone closes. Generating one report takes 3-4 hours, and delays in reporting have triggered two federal audit findings in the past 18 months for failure to meet FHWA reporting windows.

## Data model

| Table | Key fields | Purpose |
|-------|-----------|---------|
| Projects | project_name, federal_project_number, total_budget, spent_to_date, program_manager, status | Capital improvement projects receiving federal funds |
| Milestones | project_id, milestone_name, sequence, planned_completion, actual_completion, status, inspector, sign_off_files, inspection_photos | Individual deliverables within a project |
| Funding Reports | project_id, milestone_id, report_file, submitted_to, submitted_date, status | Track generated reports and submission status |

## Example prompts

- "When a milestone is marked Complete, pull the project details, generate a federal progress report PDF from our template, attach it to the milestone record, and email it to the program manager for review."
- "Automate the FHWA milestone report: as soon as an engineer marks a milestone done, build the report, file it, and notify the grants coordinator."

## Workflow

**Trigger:** `record_updated` — a milestone record's status field changes to "Complete".

```json
[
  {
    "id": "get_milestone",
    "type": "tool_call",
    "description": "Read the full milestone record including attachments",
    "tool_name": "get_record",
    "input": {
      "table_id": "milestones",
      "record_id": "{{trigger.record_updated.record_id}}"
    }
  },
  {
    "id": "get_project",
    "type": "tool_call",
    "description": "Look up the parent project for federal identifiers and budget data",
    "tool_name": "get_record",
    "input": {
      "table_id": "projects",
      "record_id": "{{get_milestone.output.project_id}}"
    }
  },
  {
    "id": "count_milestones",
    "type": "tool_call",
    "description": "Aggregate milestone completion statistics for the project",
    "tool_name": "aggregate_records",
    "input": {
      "table_id": "milestones",
      "filters": {
        "project_id": "{{get_milestone.output.project_id}}"
      },
      "aggregations": [
        { "field": "status", "function": "count" }
      ],
      "group_by": "status"
    }
  },
  {
    "id": "generate_report_pdf",
    "type": "tool_call",
    "description": "Generate the federal progress report PDF from the HTML template",
    "tool_name": "generate_pdf_from_html_template",
    "input": {
      "template_id": "fhwa_milestone_report",
      "data": {
        "federal_project_number": "{{get_project.output.federal_project_number}}",
        "project_name": "{{get_project.output.project_name}}",
        "total_budget": "{{get_project.output.total_budget}}",
        "spent_to_date": "{{get_project.output.spent_to_date}}",
        "milestone_name": "{{get_milestone.output.milestone_name}}",
        "milestone_sequence": "{{get_milestone.output.sequence}}",
        "planned_completion": "{{get_milestone.output.planned_completion}}",
        "actual_completion": "{{get_milestone.output.actual_completion}}",
        "inspector": "{{get_milestone.output.inspector}}",
        "completion_summary": "{{count_milestones.output.results}}"
      }
    }
  },
  {
    "id": "attach_report_to_milestone",
    "type": "tool_call",
    "description": "Attach the generated PDF to the milestone record",
    "tool_name": "add_files_to_record",
    "input": {
      "table_id": "milestones",
      "record_id": "{{trigger.record_updated.record_id}}",
      "files": ["{{generate_report_pdf.output.file_url}}"]
    }
  },
  {
    "id": "create_report_log",
    "type": "tool_call",
    "description": "Create a funding report record for audit trail",
    "tool_name": "create_record",
    "input": {
      "table_id": "funding_reports",
      "data": {
        "project_id": "{{get_milestone.output.project_id}}",
        "milestone_id": "{{trigger.record_updated.record_id}}",
        "report_file": "{{generate_report_pdf.output.file_url}}",
        "status": "Pending Review"
      }
    }
  },
  {
    "id": "notify_program_manager",
    "type": "tool_call",
    "description": "Notify the program manager that the report is ready for review and submission",
    "tool_name": "send_notification",
    "input": {
      "user_email": "{{get_project.output.program_manager}}",
      "title": "Milestone report ready: {{get_milestone.output.milestone_name}}",
      "message": "The FHWA progress report for {{get_project.output.project_name}} - {{get_milestone.output.milestone_name}} has been generated and attached. Please review and submit to the federal portal."
    }
  }
]
```

## Results

| Metric | Before | After |
|--------|--------|-------|
| Time to generate milestone report | 3-4 hours | Under 5 minutes |
| Federal audit findings for late reporting | 2 in 18 months | 0 |
| Reports missing required data fields | 10% | 0% (template enforced) |
| Program manager hours on reporting per month | 30 hours | 3 hours (review and submit) |

-> [Set up this workflow on Lotics](https://lotics.ai)
