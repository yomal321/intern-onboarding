# LearnLanka — Comprehensive Project Report & Requirements

## 1. Problem Statement

In Sri Lanka, the "education gap" is driven by geography; top-tier tutors for specialized subjects like A/L Physics or Chemistry are concentrated in urban hubs like Colombo or Kandy. Students in rural provinces face high travel costs and time waste to access these experts. Conversely, qualified teachers struggle to manage a professional tutoring business, often dealing with late student payments and scheduling chaos. LearnLanka is a digital marketplace designed to bridge this gap, providing a secure, vetted environment for 1-on-1 video education with integrated local payment (PayHere) and banking (Sampath Vishwa) systems.

## 2. Personas

| Persona                       | Goals                                                                                            | Frustrations                                                                                 |
| :---------------------------- | :----------------------------------------------------------------------------------------------- | :------------------------------------------------------------------------------------------- |
| **Student: Kasun (17)**       | Find a high-quality A/L Physics tutor who teaches in Sinhala and fits his budget.                | Spends 3 hours a day traveling to physical classes; lacks choices in his local village.      |
| **Tutor: Ms. Priyanthi (34)** | Increase income by teaching students nationwide and receive reliable, automated weekly payments. | Tracking manual bank transfers and losing money due to last-minute student cancellations.    |
| **Ops Admin: Tharindu (28)**  | Maintain a high standard of education on the platform through rigorous tutor vetting.            | The manual effort required to verify certificates and handle disputes over technical issues. |

## 3. Functional Requirements

### **Group 1: Students**

1.  Filter tutors by subject, grade, language, and price per hour.
2.  Book 1-hour sessions using an interactive availability calendar.
3.  Execute secure payments via PayHere (Credit Card, eZ Cash, or M-Cash).
4.  Join 1-on-1 virtual classrooms directly via a browser link.
5.  Provide a star rating and textual feedback for completed sessions.

### **Group 2: Tutors**

6.  Configure and update weekly availability time-slots.
7.  Upload NIC and educational qualification documents for verification.
8.  Access a financial dashboard showing gross earnings and net payouts.
9.  Receive real-time SMS alerts for new bookings and cancellations.

### **Group 3: Operations Admin**

10. Review, approve, or reject tutor applications via an internal admin panel.
11. Generate and export weekly payout instruction files for Sampath Vishwa banking.
12. Review flagged sessions and manually trigger student refunds.

## 4. Non-Functional Requirements

| Category         | Metric    | Target      | How we'll measure it                                       |
| :--------------- | :-------- | :---------- | :--------------------------------------------------------- |
| **Performance**  | Latency   | < 800ms     | Server-side logging of search query execution time.        |
| **Security**     | Integrity | JWT / HTTPS | Audit of API headers to ensure all requests are tokenized. |
| **Availability** | Uptime    | 99.5%       | Monthly monitoring report from Azure Application Insights. |
| **Compliance**   | Privacy   | PDPA 2022   | Verification of data encryption and user consent flags.    |

## 5. Ambiguity Analysis & Clarification (10 Items)

| ID  | Ambiguity Found         | Why it is Ambiguous                            | Clarification Question                                                                                 | Priority |
| :-- | :---------------------- | :--------------------------------------------- | :----------------------------------------------------------------------------------------------------- | :------- |
| 1   | **"Weekly Payouts"**    | Is it a specific day or a 7-day rolling cycle? | Should the bank file be generated every Friday at 5 PM for all tutors?                                 | **High** |
| 2   | **"Vetted Tutors"**     | Vague standard for approval.                   | What are the mandatory documents (e.g., NIC + Degree) required before the 'Approve' button is enabled? | **High** |
| 3   | **"Refund Policy"**     | No definition of "late" cancellation.          | If a student cancels 6 hours before, does the tutor still get 50% of the fee?                          | **High** |
| 4   | **"15% Commission"**    | Doesn't account for gateway fees.              | Does the 15% include the 3% PayHere transaction fee, or is the tutor's net 85% - 3%?                   | **High** |
| 5   | **"Video Session End"** | Hard-stop vs. Soft-stop.                       | Should the Daily.co room automatically disconnect at 60 mins, or is there a 5-minute grace period?     | **Med**  |
| 6   | **"SMS Alerts"**        | "Urgent" is not defined.                       | Is every booking change an SMS, or just those occurring within 24 hours of the start time?             | **Low**  |
| 7   | **"Search Ranking"**    | "High Quality" isn't a sort order.             | Should the default search view rank by rating, price (low-high), or tutor response time?               | **Med**  |
| 8   | **"Dispute Window"**    | Time limit to complain.                        | How many hours after a session ends does a student have to click the "Report Issue" button?            | **High** |
| 9   | **"Cloud Service"**     | Usage costs.                                   | Are there limits on the size/resolution of tutor certificate uploads to AWS/Azure storage?             | **Low**  |
| 10  | **"Identity Token"**    | Expiry time for JWT.                           | How long should a user stay logged in (JWT TTL) before being forced to re-authenticate?                | **Med**  |

## 6. Assumptions

1.  **Connectivity:** Both parties have at least 2Mbps stable internet for video calls.
2.  **Banking:** The platform uses Sampath Vishwa bulk-transfer format for payouts.
3.  **Vetting:** Manual review by the Admin is sufficient for launch; no automated government database check.
4.  **Currency:** All financial calculations are fixed in Sri Lankan Rupees (LKR).

## 7. Out of Scope

1.  **Mobile App:** Initial launch is web-only; no Android/iOS apps.
2.  **Group Classes:** System only supports 1-on-1 tutoring.
3.  **Session Recording:** Lessons are not recorded to save on storage costs.
4.  **Automatic Invoicing:** Formal PDF tax invoices are not generated in this version.

## 8. Reflection: The Cost of Ambiguity

Ambiguity is the most expensive "bug" in software engineering because it leads to wasted development time. For example, if I assume the 15% commission covers transaction fees but the business intended it to be extra, we would have to rewrite the entire payment and payout logic. Fixing a misunderstood requirement in the code can cost 10x more than fixing it during the design phase. By identifying these 10 ambiguities early, we ensure that the team builds exactly what the business needs, reducing "re-work" and ensuring the project remains on schedule and under budget.
