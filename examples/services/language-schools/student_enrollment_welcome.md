# Student Enrollment Welcome Package

## The problem

A language school enrolling 20-30 new students per month sends welcome emails manually. The admin copies a template, fills in the class schedule, teacher name, and textbook list, then sends it. Mistakes are common: wrong class time, missing Zoom link, or outdated textbook editions. New students show up confused or unprepared, and the first impression suffers. Teachers don't learn about new students until they appear in class.

## Data model

| Table | Key fields | Purpose |
|-------|-----------|---------|
| Students | full_name, email, phone, native_language, proficiency_level | Student profiles |
| Enrollments | student_id, class_id, enrollment_date, status, tuition_amount, payment_status | Enrollment records |
| Classes | class_name, language, level, teacher_id, schedule, room_or_link, textbook, max_students | Class catalog and logistics |
| Teachers | full_name, email, languages_taught | Teacher profiles |

## Example prompts

- "When a new enrollment is created, send the student a welcome email with their class schedule, teacher name, room or Zoom link, and required textbook. Also notify the teacher about the new student."
- "Automatically welcome newly enrolled students with all their class details and alert the teacher that a new student is joining."

## Workflow

**Trigger:** When an enrollment record is created

```json
[
  {
    "id": "get_student",
    "type": "tool_call",
    "description": "Fetch the student's profile",
    "tool_name": "get_record",
    "input": {
      "table_name": "Students",
      "record_id": "{{trigger.record_updated.next_data.student_id}}"
    }
  },
  {
    "id": "get_class",
    "type": "tool_call",
    "description": "Fetch class details for schedule and materials",
    "tool_name": "get_record",
    "input": {
      "table_name": "Classes",
      "record_id": "{{trigger.record_updated.next_data.class_id}}"
    }
  },
  {
    "id": "get_teacher",
    "type": "tool_call",
    "description": "Fetch the teacher's profile for contact and name",
    "tool_name": "get_record",
    "input": {
      "table_name": "Teachers",
      "record_id": "{{get_class.output.teacher_id}}"
    }
  },
  {
    "id": "welcome_email",
    "type": "tool_call",
    "description": "Send welcome email to the new student with all class details",
    "tool_name": "gmail_send_email",
    "input": {
      "to": "{{get_student.output.email}}",
      "subject": "Welcome to {{get_class.output.class_name}} - Everything you need to know",
      "body": "Hi {{get_student.output.full_name}},\n\nWelcome to {{get_class.output.class_name}}! Here's everything you need:\n\nLanguage: {{get_class.output.language}} ({{get_class.output.level}})\nTeacher: {{get_teacher.output.full_name}}\nSchedule: {{get_class.output.schedule}}\nLocation: {{get_class.output.room_or_link}}\nTextbook: {{get_class.output.textbook}}\n\nPlease have your textbook ready before your first class. If you have any questions, reply to this email.\n\nWe're excited to have you!"
    }
  },
  {
    "id": "notify_teacher",
    "type": "tool_call",
    "description": "Notify the teacher about the new student joining their class",
    "tool_name": "send_notification",
    "input": {
      "message": "New student enrolled in {{get_class.output.class_name}}: {{get_student.output.full_name}} ({{get_student.output.native_language}}, level: {{get_student.output.proficiency_level}})",
      "member_id": "{{get_class.output.teacher_member_id}}"
    }
  }
]
```

## Results

| Metric | Before | After |
|--------|--------|-------|
| Welcome email errors | 3-4/month (wrong details) | 0 (pulled from source records) |
| Admin time per enrollment | 20 min | 0 min |
| Teacher awareness of new students | Day-of (surprise) | Immediate notification |

-> [Set up this workflow on Lotics](https://lotics.ai)
