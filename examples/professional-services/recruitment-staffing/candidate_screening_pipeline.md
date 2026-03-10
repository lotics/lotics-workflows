# Candidate Screening Pipeline

## The problem

A staffing agency receives 200+ candidate applications per week across 30-40 open job orders. Recruiters manually review each resume, check for minimum qualifications, and send screening questionnaires. During peak hiring seasons, it takes 4-5 days to respond to applicants — by which time top candidates have already accepted offers elsewhere. Recruiters spend 60% of their time on administrative screening tasks instead of relationship-building and closing placements.

## Data model

| Table | Key fields | Purpose |
|-------|-----------|---------|
| Candidates | candidate_name, email, phone, resume_file, skills, experience_years, status, source, applied_date | Candidate profiles and application data |
| Job Orders | job_title, client_company, required_skills, min_experience, rate_range, status, recruiter | Open positions from client companies |
| Screening Results | candidate (link), job_order (link), match_score, qualification_notes, status, screened_at | Results of automated screening against job requirements |

## Example prompts

- "When a candidate applies via the web form, extract their resume data, match them against open job orders, score the fit, and send them a screening questionnaire if they're a potential match."
- "Automate candidate intake: parse the resume, check qualifications against all open roles, notify the recruiter of strong matches, and auto-reject candidates who don't meet minimum requirements."

## Workflow

**Trigger:** `form_submitted` — candidate submits application through the agency's website.

```json
[
  {
    "id": "create_candidate",
    "type": "tool_call",
    "description": "Create a candidate record from the application form",
    "tool_name": "create_record",
    "input": {
      "table_name": "Candidates",
      "data": {
        "candidate_name": "{{trigger.form_submitted.data.candidate_name}}",
        "email": "{{trigger.form_submitted.data.email}}",
        "phone": "{{trigger.form_submitted.data.phone}}",
        "source": "Website",
        "status": "Screening",
        "applied_date": "{{today()}}"
      }
    }
  },
  {
    "id": "attach_resume",
    "type": "tool_call",
    "description": "Attach the uploaded resume to the candidate record",
    "tool_name": "add_files_to_record",
    "input": {
      "table_name": "Candidates",
      "record_id": "{{create_candidate.output.record_id}}",
      "field_name": "resume_file",
      "file_ids": "{{trigger.form_submitted.data.resume_file_ids}}"
    }
  },
  {
    "id": "parse_resume",
    "type": "tool_call",
    "description": "Extract structured data from the resume",
    "tool_name": "extract_file_data",
    "input": {
      "file_id": "{{trigger.form_submitted.data.resume_file_ids[0]}}",
      "extraction_prompt": "Extract the following from this resume: skills (as array), total_experience_years (number), most_recent_title, education_level, certifications (as array). Return as JSON."
    }
  },
  {
    "id": "update_candidate_data",
    "type": "tool_call",
    "description": "Update the candidate record with parsed resume data",
    "tool_name": "update_record",
    "input": {
      "table_name": "Candidates",
      "record_id": "{{create_candidate.output.record_id}}",
      "data": {
        "skills": "{{parse_resume.output.extracted.skills}}",
        "experience_years": "{{parse_resume.output.extracted.total_experience_years}}"
      }
    }
  },
  {
    "id": "get_open_jobs",
    "type": "tool_call",
    "description": "Fetch all open job orders to match against",
    "tool_name": "query_records",
    "input": {
      "table_name": "Job Orders",
      "filters": {
        "status": "Open"
      }
    }
  },
  {
    "id": "score_matches",
    "type": "tool_call",
    "description": "Use LLM to score the candidate against all open job orders",
    "tool_name": "llm_generate_text",
    "input": {
      "prompt": "Score this candidate against each open job order. For each job, provide a match_score (0-100) and brief qualification_notes.\n\nCandidate:\n- Skills: {{JSON.stringify(parse_resume.output.extracted.skills)}}\n- Experience: {{parse_resume.output.extracted.total_experience_years}} years\n- Recent Title: {{parse_resume.output.extracted.most_recent_title}}\n- Education: {{parse_resume.output.extracted.education_level}}\n- Certifications: {{JSON.stringify(parse_resume.output.extracted.certifications)}}\n\nOpen Job Orders:\n{{JSON.stringify(get_open_jobs.output.records)}}\n\nReturn as JSON array with fields: job_order_id, job_title, match_score, qualification_notes. Only include jobs with match_score >= 40."
    }
  },
  {
    "id": "check_any_matches",
    "type": "if",
    "description": "Check if the candidate matched any job orders",
    "condition": "{{JSON.parse(score_matches.output.text).length > 0}}",
    "then": [
      {
        "id": "create_screening_results",
        "type": "foreach",
        "description": "Create screening result records for each matched job order",
        "items": "{{JSON.parse(score_matches.output.text)}}",
        "steps": [
          {
            "id": "create_result",
            "type": "tool_call",
            "description": "Save the screening result for this job match",
            "tool_name": "create_record",
            "input": {
              "table_name": "Screening Results",
              "data": {
                "candidate": "{{create_candidate.output.record_id}}",
                "job_order": "{{item.job_order_id}}",
                "match_score": "{{item.match_score}}",
                "qualification_notes": "{{item.qualification_notes}}",
                "status": "Pending Review",
                "screened_at": "{{now()}}"
              }
            }
          }
        ]
      },
      {
        "id": "update_candidate_status",
        "type": "tool_call",
        "description": "Update candidate status to indicate matches found",
        "tool_name": "update_record",
        "input": {
          "table_name": "Candidates",
          "record_id": "{{create_candidate.output.record_id}}",
          "data": {
            "status": "Matched"
          }
        }
      },
      {
        "id": "send_acknowledgement",
        "type": "tool_call",
        "description": "Send the candidate a confirmation with next steps",
        "tool_name": "gmail_send_email",
        "input": {
          "to": "{{trigger.form_submitted.data.email}}",
          "subject": "Application Received — We found matching opportunities",
          "body": "Hi {{trigger.form_submitted.data.candidate_name}},\n\nThank you for applying. We've reviewed your qualifications and identified potential matches with our open positions.\n\nA recruiter will contact you within one business day to discuss the opportunities in more detail.\n\nBest regards,\nRecruitment Team"
        }
      }
    ],
    "else": [
      {
        "id": "update_no_match",
        "type": "tool_call",
        "description": "Update candidate status to indicate no current matches",
        "tool_name": "update_record",
        "input": {
          "table_name": "Candidates",
          "record_id": "{{create_candidate.output.record_id}}",
          "data": {
            "status": "No Current Match"
          }
        }
      },
      {
        "id": "send_no_match_email",
        "type": "tool_call",
        "description": "Send a polite email indicating no current openings match",
        "tool_name": "gmail_send_email",
        "input": {
          "to": "{{trigger.form_submitted.data.email}}",
          "subject": "Application Received — {{trigger.form_submitted.data.candidate_name}}",
          "body": "Hi {{trigger.form_submitted.data.candidate_name}},\n\nThank you for your interest. We've reviewed your qualifications and unfortunately don't have a matching opening at this time.\n\nWe've added you to our talent pool and will reach out when a suitable position becomes available.\n\nBest regards,\nRecruitment Team"
        }
      }
    ]
  }
]
```

## Results

| Metric | Before | After |
|--------|--------|-------|
| Candidate response time | 4-5 days | < 30 minutes |
| Recruiter time on screening | 60% of day | 20% of day |
| Candidate drop-off rate | 35% | 12% |

-> [Set up this workflow on Lotics](https://lotics.ai)
