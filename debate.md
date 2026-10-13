
# SECTION - 1 

A. Assesment questions
1. we have two options, admin creates assesment that will contain sections and that assesment will contain tasks
2. then the admin will add the candidate for this assesment this will create participant record, and asseignment with its sessions.

my cross question is that, if i want to conduct an mass assignment where mulitple participant will attend the assignment,  how i'm going to make sure that assesment , assignemt for all of them is unique to preven similar questions for each particant, 

if i assume each participant will have their own assignment record,  then one way to do this is, the interested participants will fill out the form and that data will be extracted and program will automatically generate assignment for each participant, then will create particapant record, then the assesment and a command configured assignment session.  


Now review my approch and suggest any improvenent. 

# Section - 2 

### understanding the flow through the ER diagram 
A. . i'm focusing on Assesment, assingment, task, assesment_session, session task/answer, submissions, results section.

this is my approch to the flow
1. admin creates assessment, this assesment will contain sections and each sections will contain task pooling configuration. this will define how a assesment session will create dymanic and randomised tasks for multiple candidates.
2. admin creates participant and for that assesment and assignment for that participant.
3. the link will be shared to the participant, and let assume the part.. logins 
4. clicks start , this will check if the part x is allowed to take assesment y, with the help of assignment the admin has created for this part.. .
5. if it allowed then create the backened fetches the assesment which has the configuration for the task pooling.
6. using the configuration, assesment_session will be created ( this is what will be monitered in proctoring).
7. now the assesment_session has been created the server creates session tasks for this assesment_session and the task will be send to the participant  ( till here we have generated random tasks for each participant). 
8. now the participant attends it, and at last submits the submission.  this will create a submission for the assesment_session. 
9. once the submission is created, the evaluation is done and the results are generated. 


### now this is my approch for system flow, now tell me missing peaces in it. exact flow which follows my strategy for this , what you have to focus is on erd i'm mentioning to you and entities related to the it.

# Section - 3 
[updated_project_erd.png](file;file:///c%3A/Users/neeteshp/Documents/project_proposal/updated-proposal-db-design/updated_project_erd.png) 
1. there are entities in this ERD
2. explain each entity one by one with its attributes and reason for taking the attribute.

# Section - 4 ( ERD-Assesment-section ) 

@file:updated_erd_question_pooling.png
i will ask cross questions about the ERD in order to make a final implementable versionnnnnnn so think carefully in the context
you can take the context from
@file:debate.md  @file:deabte-answers.md
1.
Assignement setup  ERD section
A. assessments
1.
why we need previous version here
2.
what are we goiing to store in the settings jsonb ( is it fessible to store the proctoring settings and other setting in json )
B. section_pool_rules
3.
why are we not storing assestment id here too with section_Id. ( for optimised indexing )
4.
what we going to store in skill_filter jso

# Section -5  ( delivery section of ERD ) 

ok, the  assesment section of ERD is clear,
the next Section of ERD is Delivery which contains, participants, assignment, assesment_session, answer / session task ) and submissions
I'm mentioning the attribute i have doubt in and need clear cut picture with what i'm goinng to work with

@file:updated_erd_question_pooling-version-2.0.png
A. Participants
1. why dept attribute?
2. why refrence no and what is it.
3. participant type ?

B. Assignments
1.  how status will look alike
2. what is UQ stand for

C. assesment_sessions
1. what is submit type
2. Status ?

D. Answers / session tasks  ==> we will call is assessment_session_tasks

1. why pool rule id
2. why display order
3. if we storing marks and  negative marks --> why to store them when we are configuring task marks by section
4. why answer json is there and what it will look like.
5. answer status
6.  eval status
7. llm evaluation jsonb
8. why feed back is here ?


E. Submission --> we will call it assessment_seesion_task_submission

1. status ? what will it define
2. verdict ?
3. what is test results jsonb and why is there, we are also storing some similar attributes to session_tasks


# Section - 6  ( Acces section ERD )

ok greate, the delivery part of the ERD is clear

now comes the Access section of ERD
1.  users is fine, but how and why its related to participant.
2. in the roles what and how the permission will look alike 

# Section - 7 ( Outcome  section ERD ) 

great, the Access part is done

now comes the Outcome section of ERD

A. Results
1.  status  ?
2. section score jsonb
3. integraty summy  jsonb
4. integrity flagged

B.  how the skill scores will be used 



# Section - 8 ( Monitoring ) 

Excelent, now this part is clear to me
now comes    Monitoring part

A. proctoring events
1.  event_type --> what it is?
2. what is details jsonb

B. Audit_logs
1.  explain why this table is exists
2. explain each attribute perpous and how it will look like 


# Section - 9 ( Question Section ERD ) 

amazing, the monitoring part is also clear

now the last section and important is question bank section

A. Tasks

1.  skills json ?
2. content json ?
3. status ?
4. default marks --> i think its  not needed --> we can mark task by section
5. why prev_task Id is maintained

B. skills --> How it will be used

C. test cases
1. weight ?
2. is_performance --> what and why

D. task options --> what it is
1. option_text
2. is_correct
3. why display order 

