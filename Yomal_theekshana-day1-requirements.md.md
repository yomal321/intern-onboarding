# LearnLanka — Requirements Document (v5)

## 1. Problem Statement

In Sri Lanka, students preparing for O/L and A/L exams often struggle to find high-quality, specialized tutors outside of their immediate local area, leading to long travel times and limited choices. At the same time, qualified tutors lack a centralized platform to manage bookings and secure payments reliably. LearnLanka solves this by providing a digital marketplace that bridges the gap, allowing students to access vetted education from home while ensuring tutors are paid fairly and on time.

## 2. Personas

### **Student: Kasun (17, A/L Student from Galle)**

- **Goal:** Find a high-quality Physics tutor who teaches in Sinhala and fits his after-school schedule.
- **Frustration:** Spends 2 hours traveling to tuition classes and finds it difficult to ask questions in large physical group classes.

### **Tutor: Ms. Priyanthi (34, Experienced Math Teacher)**

- **Goal:** Increase her monthly income by teaching students from across the country without leaving her home.
- **Frustration:** Dealing with students who cancel at the last minute without paying and the hassle of manually tracking bank transfers for every individual session.

### **Operations Admin: Tharindu (28, LearnLanka Staff)**

- **Goal:** Ensure only qualified tutors are active on the platform and resolve disputes between students and tutors quickly.
- **Frustration:** Manually verifying tutor certificates and handling complaints about technical issues during video calls.

## 3. Functional Requirements

### **Group 1: Students**

1.  The system shall allow students to filter tutors by subject, grade level, language, and price.
2.  The system shall allow students to book specific time slots from a tutor's calendar.
3.  The system shall process payments via PayHere (Card or eZ Cash).
4.  The system shall allow students to submit ratings and reviews.
5.  The system shall send automated SMS alerts for booking confirmations.

### **Group 2: Tutors**

6.  The system shall allow tutors to manage their availability for sessions.
7.  The system shall provide a dashboard to accept/decline booking requests.
8.  The system shall show tutors their weekly earnings after platform commission.
9.  The system shall send SMS notifications for new booking requests.

### **Group 3: Operations Admin**

10. The system shall allow admins to verify and approve tutor profiles.
11. The system shall generate weekly payout reports for bank transfers.
12. The system shall allow admins to trigger manual SMS alerts for system updates.

## 4. External Systems (In Scope for Context Diagram)

- **Payment Gateway (PayHere):** Processes student payments.
- **Video Service (Daily.co / 100ms):** Facilitates real-time 1-on-1 video lessons.
- **SMS Gateway (ShoutOUT / FITSMS):** Delivers OTPs and session alerts to mobile phones.
- **Banking System (Sampath Vishwa):** Executes weekly automated payouts to tutors.
- **Cloud Service (AWS/Azure):** Used for storage of media, certificates, and system data.

## 5. Non-Functional Requirements

- **Performance:** Search results must load within 800ms.
- **Security:** All communications between Users, Central System, and External APIs must use **HTTPS**.
- **Integrity:** Payout data must match session records exactly.
- **Availability:** System must be available 99.5% of the time for bookings.

## 6. Diagram Legend (Notation Explanation)

- **Oval:** User Persona (External Actor).
- **Solid Rectangle:** System in Scope (The LearnLanka Platform).
- **Dashed Rectangle:** External System (Third-party services).
- **Arrows:** Represent data flow and interaction using **HTTPS** and **REST APIs**.

## 7. Out of Scope

- Native Mobile Applications (Initial launch is Web-based).
- Group class functionality (1-to-1 only).
- Learning Management System (LMS) tools like quizzes and homework.
