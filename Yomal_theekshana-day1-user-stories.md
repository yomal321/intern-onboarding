# LearnLanka — User Stories & Requirements Document (8 Stories)

## 1. Problem Statement

Sri Lankan students in rural areas face a significant "geographic barrier" to quality education, often traveling hours to attend tuition classes in urban centers. Conversely, many talented teachers lack a secure way to reach these students and handle financial transactions without the risk of non-payment. LearnLanka bridges this gap by providing a localized marketplace that facilitates vetted tutor discovery, secure card/mobile payments, and integrated video classrooms, ensuring that high-quality education is accessible to any student with an internet connection.

## 2. Personas

| Persona                   | Goals                                                                          | Frustrations                                                                                        |
| :------------------------ | :----------------------------------------------------------------------------- | :-------------------------------------------------------------------------------------------------- |
| **Student (Kasun)**       | Find specialized A/L Physics tutors who teach in Sinhala at affordable rates.  | Long travel times to physical classes and lack of tutor variety in his local village.               |
| **Tutor (Ms. Priyanthi)** | Earn extra income by teaching remotely and receive guaranteed weekly payments. | Students canceling last minute without paying and the difficulty of tracking manual bank transfers. |
| **Ops Admin (Tharindu)**  | Ensure platform safety by vetting tutors and resolving disputes efficiently.   | The high manual workload of checking physical certificates and handling student complaints.         |

## 3. User Stories & Acceptance Criteria

### Story 1: Tutor Discovery (Student)

**As a** Student,  
**I want to** filter tutors by subject, language, and hourly rate,  
**so that** I can find a teacher who fits my specific exam needs and my family's budget.

- **Acceptance Criteria:**
  - **Given** I am on the tutor listing page, **When** I select "Mathematics" and "Sinhala", **Then** the list should update to show only tutors teaching those options.
  - **Given** a list of filtered tutors, **When** I set the price range to "1000 - 2000 LKR", **Then** any tutor charging more than 2000 LKR should be hidden.
- **INVEST Self-Check:**
  - [x] **I**ndependent | [x] **N**egotiable | [x] **V**aluable | [x] **E**stimable | [x] **S**mall | [x] **T**estable
  - _Note: All criteria met._

### Story 2: Performance & User Retention (Non-Functional)

**As a** Student,  
**I want** the tutor search results to load almost instantly,  
**so that** I don't experience frustration or leave the platform due to slow performance on my mobile connection.

- **Acceptance Criteria:**
  - **Given** I am using a standard 4G mobile connection in Sri Lanka, **When** I click the "Search" button, **Then** the results must be visible on screen within 800 milliseconds.
- **INVEST Self-Check:**
  - [x] **I**ndependent | [x] **N**egotiable | [x] **V**aluable | [x] **E**stimable | [ ] **S**mall | [x] **T**estable
  - _Note: Performance tuning is often a large, open-ended task (not "Small") as it involves database indexing and API optimization._

### Story 3: Availability Management (Tutor)

**As a** Tutor,  
**I want to** toggle my free hourly slots on a digital calendar,  
**so that** I don't receive booking requests during my personal time or school teaching hours.

- **Acceptance Criteria:**
  - **Given** I am on my availability dashboard, **When** I click a time slot (e.g., Monday 4:00 PM), **Then** the slot should change color to indicate I am "Available."
  - **Given** a student is viewing my profile, **When** they look at my calendar, **Then** only the slots I marked as "Available" should be clickable for booking.
- **INVEST Self-Check:**
  - [x] **I**ndependent | [x] **N**egotiable | [x] **V**aluable | [x] **E**stimable | [x] **S**mall | [x] **T**estable
  - _Note: All criteria met._

### Story 4: Earnings Transparency (Tutor)

**As a** Tutor,  
**I want to** view a clear breakdown of my pending and completed earnings,  
**so that** I can verify that the 15% platform commission is calculated correctly.

- **Acceptance Criteria:**
  - **Given** I have completed a 2000 LKR lesson, **When** I view my "Earnings" tab, **Then** I should see a line item showing "1700 LKR (Net)" and "300 LKR (Commission)."
  - **Given** I have multiple sessions in a week, **When** I check the "Total Payout," **Then** it should sum all net earnings for that specific week accurately.
- **INVEST Self-Check:**
  - [x] **I**ndependent | [x] **N**egotiable | [x] **V**aluable | [x] **E**stimable | [x] **S**mall | [x] **T**estable
  - _Note: All criteria met._

### Story 5: Quality Assurance (Ops Admin)

**As an** Operations Admin,  
**I want to** review and approve tutor identity documents,  
**so that** we can prevent unqualified or fraudulent individuals from interacting with students.

- **Acceptance Criteria:**
  - **Given** a new tutor has uploaded their NIC and Degree, **When** I view the "Pending Approvals" queue, **Then** I should be able to see the high-resolution images of these documents.
  - **Given** I have reviewed a profile, **When** I click "Approve," **Then** the tutor's profile should immediately become "Active" and visible to students.
- **INVEST Self-Check:**
  - [x] **I**ndependent | [x] **N**egotiable | [x] **V**aluable | [x] **E**stimable | [x] **S**mall | [x] **T**estable
  - _Note: All criteria met._

### Story 6: Financial Disbursement (Ops Admin)

**As an** Operations Admin,  
**I want to** generate a batch payout file for the bank,  
**so that** hundreds of tutors can be paid their weekly earnings automatically.

- **Acceptance Criteria:**
  - **Given** the weekly payout cycle has ended, **When** I click "Generate Bank File," **Then** the system should create a CSV file formatted specifically for Sampath Vishwa bulk transfers.
  - **Given** a payout file is generated, **When** the bank confirms processing, **Then** the status of all included earnings should change from "Pending" to "Paid."
- **INVEST Self-Check:**
  - [x] **I**ndependent | [ ] **N**egotiable | [x] **V**aluable | [x] **E**stimable | [x] **S**mall | [x] **T**estable
  - _Note: The bank's file format is strictly defined and not "Negotiable."_

### Story 7: Real-time Lesson Alerts (Student)

**As a** Student,  
**I want to** receive an SMS reminder 15 minutes before my lesson starts,  
**so that** I don't forget to join the video call and waste my paid session time.

- **Acceptance Criteria:**
  - **Given** I have a confirmed booking for 5:00 PM, **When** the system clock reaches 4:45 PM, **Then** an automated SMS should be sent to my registered mobile number.
  - **Given** I receive the reminder SMS, **When** I click the link inside the message, **Then** I should be taken directly to the session's video waiting room.
- **INVEST Self-Check:**
  - [x] **I**ndependent | [x] **N**egotiable | [x] **V**aluable | [x] **E**stimable | [x] **S**mall | [x] **T**estable
  - _Note: All criteria met._

### Story 8: Dispute & Refund Handling (Ops Admin)

**As an** Operations Admin,  
**I want to** review flagged sessions and process refunds,  
**so that** I can maintain trust in the platform when technical issues or absences occur.

- **Acceptance Criteria:**
  - **Given** a student has clicked "Report Issue" after a failed session, **When** I log into the Admin panel, **Then** I should see the session details and the student's reason for the report.
  - **Given** a valid dispute, **When** I click "Issue Refund," **Then** the system should communicate with the Payment Gateway to return the funds to the student's original payment method.
- **INVEST Self-Check:**
  - [x] **I**ndependent | [x] **N**egotiable | [x] **V**aluable | [x] **E**stimable | [ ] **S**mall | [x] **T**estable
  - _Note: Dispute resolution is complex and often requires manual investigation, making it a large ("Not Small") task._
