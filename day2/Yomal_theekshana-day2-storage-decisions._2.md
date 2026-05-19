# BookSwap — Queue Plan

## 1. Which work goes on a queue and why

Heavy background tasks and third-party communications are pushed out to **Azure Service Bus Queues** to keep user-facing actions fast and responsive:

- **Email Notification Dispatch Tasks:** Operations like building the **Weekly 10-Most-Recent-Books Digest** or sending out transactional email alerts run asynchronously. The backend processes the core operation instantly, logs it to the database, and drops a small task message onto the queue before immediately returning a success code to the user. This ensures book listings succeed even if the email system is down.
- **Real-Time Push-Alert Trimming:** Offloading push notification formatting and delivery tasks to background queue workers guarantees that in-app alerts hit owners' devices within the 2-second target without blocking the main application thread.

## 2. Failure Mode Analysis: Consumer down for 30 minutes

If the background email consumer service drops offline or loses its connection to Azure Communication Services for 30 minutes, message handling is managed through this built-in resilience framework:

- **Message Retention Security (No Data Loss):** The queue acts as a temporary durable storage buffer. Because messages are persistent on disk, they sit safely in the queue during an outage rather than being lost in volatile memory. The main web application can continue creating book listings and accepting borrow requests without throwing errors to the user.
- **Catch-Up Processing:** As soon as the background worker comes back online, it resumes pulling tasks from the queue in order. The worker can catch up on backlogged tasks without losing data or overloading the email service.
- **Retry Engine and Dead-Letter Queue (DLQ) Isolation:** If a worker fails to process a message due to an unhandled exception or data error, the queue initiates a built-in retry workflow:
  1. The queue retries processing up to **3 consecutive times** with an exponential backoff delay.
  2. If the message fails all 3 attempts, the engine moves it into a separate **Dead-Letter Queue (DLQ)**.  
     By isolating failing or corrupt messages (poison pills) in the DLQ, the system prevents them from blocking the rest of the queue, allowing standard messages to process normally. Engineers can then review the DLQ separately to diagnose issues.
