# **Product Requirements Document**

## **Email Broadcast System**

**Module:** Platform Administration **Product:** JudicAI

| Field | Detail |
| :---- | :---- |
| Product | JudicAI |
| Module | Platform Administration |
| Feature Type | Administrative Communication |
| Priority | High |
| Status | Proposed |
| Prepared By | Product Team |

## **01 — Overview**

The Email Broadcast System is a Platform Admin feature that enables the Super Admin to create, schedule, manage, and send email communications to specific categories of users on the JudicAI platform.

The system supports targeted messaging based on user roles, court affiliations, subscription plans, and geographic locations. The objective is to centralise all platform communications within JudicAI, eliminating reliance on external email tools.

**Architecture Note:** This feature is designed as the first channel of what should eventually become a broader Communication Centre, capable of supporting Email, SMS, WhatsApp, In-App Notifications, and Push Notifications without requiring a full rebuild. Naming the module accordingly from the start will reduce future engineering overhead as JudicAI scales.

## **02 — Problem Statement**

Platform communications currently include:

* Product updates and new feature announcements  
* Scheduled maintenance notifications  
* Training invitations and deployment notices  
* Subscription renewal reminders

All of these are sent manually using external email platforms, resulting in:

* Operational inefficiency from context-switching between tools  
* Inconsistent communication across courts and user groups  
* No ability to track delivery or engagement  
* Limited and entirely manual audience targeting

## **03 — Goals**

### **Primary Goals**

* Allow the Super Admin to send emails directly from JudicAI  
* Support targeted audience selection using multiple filter criteria  
* Track delivery and engagement for every broadcast  
* Maintain a searchable communication history

### **Secondary Goals**

* Improve user engagement across courts  
* Increase adoption of newly released features  
* Support court-specific and division-level communications  
* Provide lightweight analytics per broadcast

## **04 — Scope**

### **In Scope (Phase 1\)**

* Broadcast creation and sending  
* Audience segmentation and filtering  
* Broadcast scheduling  
* Delivery and engagement analytics  
* Reusable email templates  
* Broadcast history and logs

### **Out of Scope (Phase 1\)**

* SMS, WhatsApp, and push notification broadcasts  
* In-app notifications  
* A/B testing  
* Automated or triggered campaigns  
* Multi-language support

## **05 — User Stories**

**Note:** All broadcast actions are performed exclusively by the Super Admin. There is no separate Admin or Viewer role for this feature. The stories below reflect the different use cases the Super Admin handles on behalf of the platform.

### **US-01 — Create and Send a Broadcast**

As a **Super Admin**, I want to create a new email broadcast and send it to a selected group of users so that I can communicate platform updates and important information efficiently without leaving JudicAI.

#### **Process Flow**

1. Super Admin navigates to Platform Admin and opens the Broadcast Dashboard.  
2. Super Admin clicks 'New Broadcast'.  
3. Super Admin fills in the Broadcast Title, Subject Line, Sender Name, and Reply-To Email.  
4. Super Admin composes the email body using the rich text editor.  
5. Super Admin optionally adds attachments (PDF, image, or document).  
6. Super Admin selects the target audience using one or more filter options.  
7. System displays the estimated recipient count based on the selected filters.  
8. Super Admin clicks 'Preview' to review the email in desktop and mobile view.  
9. Super Admin clicks 'Send Now'.  
10. System confirms the action and dispatches the broadcast.  
11. Broadcast status updates to 'Sending', then 'Sent' on completion.  
12. System logs the action in the Audit Trail.

#### **Acceptance Criteria**

| \# | Criteria |
| ----- | ----- |
| AC01 | The Create Broadcast form captures all required fields: Broadcast Title, Subject Line, Sender Name, and Reply-To Email. The form cannot be submitted if any required field is empty. |
| AC02 | The rich text editor supports formatted text, headings, lists, embedded images, links, buttons, and tables. |
| AC03 | The Super Admin can attach files (PDF, image, document) before sending. Unsupported file types are rejected with an error message. |
| AC04 | Audience selection supports all filter options: Everyone, User Role, Court, Subscription Plan, Custom Selection, and combined filters. |
| AC05 | The system displays an accurate estimated recipient count immediately after the Super Admin applies audience filters. |
| AC06 | The broadcast preview renders correctly in both desktop and mobile view before dispatch. |
| AC07 | Clicking 'Send Now' triggers an immediate dispatch and updates the broadcast status to 'Sending'. |
| AC08 | Once delivery is complete, the broadcast status updates to 'Sent'. If delivery fails for any recipients, the status updates to 'Failed' and error details are accessible. |
| AC09 | Every send action is captured in the Audit Trail with the Super Admin ID, action type, timestamp, and Broadcast ID. |

### **US-02 — Schedule a Broadcast**

As a **Super Admin**, I want to schedule a broadcast for a future date and time so that I can plan communications in advance and ensure they reach users at the right moment without manual intervention.

#### **Process Flow**

1. Super Admin creates a broadcast and completes all required fields.  
2. Super Admin selects the target audience.  
3. Instead of 'Send Now', Super Admin selects 'Schedule'.  
4. Super Admin picks a future date and time from the date-time picker.  
5. Super Admin confirms the schedule.  
6. Broadcast status updates to 'Scheduled'.  
7. System displays the scheduled broadcast on the Broadcast Dashboard with its dispatch date and time.  
8. At the scheduled time, the system automatically dispatches the broadcast.  
9. Broadcast status updates to 'Sending', then 'Sent' on completion.  
10. System logs the schedule and send actions in the Audit Trail.

#### **Acceptance Criteria**

| \# | Criteria |
| ----- | ----- |
| AC01 | The scheduling option is available on the send step of the Create Broadcast flow. |
| AC02 | The date-time picker does not allow selection of a past date or time. Attempting to schedule in the past surfaces a validation error. |
| AC03 | On confirmation, the broadcast status updates to 'Scheduled' and is visible on the Broadcast Dashboard with the correct dispatch date and time displayed. |
| AC04 | The system automatically dispatches the broadcast at the scheduled time without any manual trigger. |
| AC05 | If the Super Admin cancels a scheduled broadcast before dispatch, the status updates to 'Cancelled' and the broadcast is no longer sent. |
| AC06 | Both the scheduling action and the eventual send action are logged separately in the Audit Trail. |

### **US-03 — Save a Broadcast as Draft**

As a **Super Admin**, I want to save an in-progress broadcast as a draft so that I can return to complete and send it later without losing any of my work.

#### **Process Flow**

1. Super Admin begins creating a broadcast and fills in one or more fields.  
2. Super Admin clicks 'Save as Draft'.  
3. System saves the broadcast with all completed fields intact.  
4. Broadcast appears on the Broadcast Dashboard under 'Draft' status.  
5. Super Admin later returns to the dashboard, locates the draft, and clicks 'Edit'.  
6. Super Admin completes remaining fields and either sends immediately or schedules the broadcast.

#### **Acceptance Criteria**

| \# | Criteria |
| ----- | ----- |
| AC01 | 'Save as Draft' is available at any point during the Create Broadcast flow, even if only the Broadcast Title has been filled in. |
| AC02 | On save, all completed fields are persisted exactly as entered. No data is lost. |
| AC03 | The draft appears on the Broadcast Dashboard under the 'Draft' status counter and in the broadcast list. |
| AC04 | The Super Admin can re-open, edit, and complete a draft at any time before sending. |
| AC05 | A draft can be deleted from the dashboard. Deletion requires a confirmation prompt before the record is removed. |
| AC06 | Draft creation is logged in the Audit Trail. |

### **US-04 — Target a Specific Audience**

As a **Super Admin**, I want to filter broadcast recipients by role, court, subscription plan, or a combination of these so that I can send relevant, targeted communications to the right users instead of broadcasting to everyone.

#### **Process Flow**

1. Super Admin reaches the Audience Selection step during broadcast creation.  
2. Super Admin chooses one of the available filter options: Everyone, User Role, Court, Subscription Plan, Custom Selection, or Combined Filters.  
3. Super Admin applies the relevant filter values (e.g. selects 'Registrar' under User Role, then 'Kaduna' under State).  
4. System updates the estimated recipient count in real time as filters are applied.  
5. Super Admin reviews the count and proceeds to preview and send.

#### **Acceptance Criteria**

| \# | Criteria |
| ----- | ----- |
| AC01 | The Audience Builder supports all six filter options: Everyone, User Role, Court-Based, Subscription Plan, Custom Selection, and Combined Filters. |
| AC02 | User Role filter includes all roles: Judges, Magistrates, Registrars, Clerks, Lawyers, Court Administrators, and ICT Officers. |
| AC03 | Court-Based filter supports filtering by specific court, court type (e.g. High Court, Magistrate Court), state, and judicial division. |
| AC04 | Subscription Plan filter supports all four plan tiers: Basic, Standard, Essential, and Pro. |
| AC05 | Custom Selection allows the Super Admin to search for and manually add individual users to the recipient list. |
| AC06 | Combined Filters allow multiple criteria to be applied simultaneously. The system applies all active filters with AND logic. |
| AC07 | The estimated recipient count updates in real time as filters are added or removed. |
| AC08 | Sending a broadcast to an empty audience (zero recipients after filtering) is blocked. The system displays an error and prevents dispatch. |

### **US-05 — Preview a Broadcast Before Sending**

As a **Super Admin**, I want to preview how my broadcast will appear to recipients before I send it so that I can catch formatting issues or errors and ensure the email looks correct on both desktop and mobile.

#### **Process Flow**

1. Super Admin completes the broadcast content and audience selection.  
2. Super Admin clicks 'Preview'.  
3. System renders a preview of the email as it will appear to recipients.  
4. Super Admin toggles between Desktop and Mobile preview modes.  
5. Super Admin reviews the content, formatting, and layout.  
6. Super Admin either returns to edit or proceeds to send/schedule.

#### **Acceptance Criteria**

| \# | Criteria |
| ----- | ----- |
| AC01 | The preview is accessible before any send or schedule action is taken. |
| AC02 | The preview renders the exact content composed in the rich text editor, including formatting, images, links, and buttons. |
| AC03 | Desktop and mobile preview modes are available and the toggle between them is immediate. |
| AC04 | The preview accurately reflects how the email will render to recipients. |
| AC05 | The Super Admin can exit the preview and return to the editor to make changes without losing any content. |

### **US-06 — Use and Manage Broadcast Templates**

As a **Super Admin**, I want to create and reuse email templates for recurring communication types so that I can speed up broadcast creation and maintain consistency in platform communications.

#### **Process Flow**

1. Super Admin navigates to the Template Library from the Broadcast Dashboard.  
2. Super Admin views available system templates and any previously saved custom templates.  
3. Super Admin selects a template to use as the starting point for a new broadcast.  
4. System pre-populates the Create Broadcast form with the template content.  
5. Super Admin edits the content as needed and proceeds to send.  
6. Alternatively, Super Admin creates a new custom template by composing content and clicking 'Save as Template'.

#### **Acceptance Criteria**

| \# | Criteria |
| ----- | ----- |
| AC01 | The Template Library is accessible from the Broadcast Dashboard. |
| AC02 | Six system templates are available by default: Feature Release, Maintenance Notice, Subscription Reminder, Deployment Notice, Training Invitation, and Security Alert. |
| AC03 | Selecting a template pre-populates the broadcast creation form with the template's content. All pre-populated fields remain fully editable. |
| AC04 | The Super Admin can save any broadcast content as a custom template with a name they define. |
| AC05 | Custom templates appear in the Template Library alongside system templates. |
| AC06 | The Super Admin can delete custom templates. A confirmation prompt is shown before deletion. |
| AC07 | System templates cannot be deleted. |
| AC08 | Template creation and deletion are logged in the Audit Trail. |

### **US-07 — Track Broadcast Performance**

As a **Super Admin**, I want to view delivery and engagement analytics for each broadcast I send so that I can understand how well my communications are landing and make informed decisions about future broadcasts.

#### **Process Flow**

1. Super Admin navigates to the Broadcast Dashboard or Broadcast History.  
2. Super Admin selects a sent broadcast to view its analytics.  
3. System displays delivery metrics: Recipients, Delivered, and Failed counts.  
4. System displays engagement metrics: Opens and Clicks.  
5. Super Admin filters the analytics breakdown by user role, court, or state.

#### **Acceptance Criteria**

| \# | Criteria |
| ----- | ----- |
| AC01 | Analytics are available for every broadcast with a 'Sent' status. |
| AC02 | Delivery metrics displayed include: total recipients targeted, successfully delivered count, and failed delivery count. |
| AC03 | Engagement metrics displayed include: unique opens and unique link clicks. |
| AC04 | The audience breakdown allows filtering of opens and engagement by user role, court, and state. |
| AC05 | Analytics data updates in near real time after dispatch and does not require a manual page refresh to reflect new data. |
| AC06 | Analytics for a broadcast are retained in Broadcast History indefinitely and accessible at any time. |

### **US-08 — View and Reuse Past Broadcasts**

As a **Super Admin**, I want to access a history of all past broadcasts and duplicate or resend them so that I can maintain a complete communication record and reuse effective broadcasts without rebuilding them from scratch.

#### **Process Flow**

1. Super Admin navigates to Broadcast History from the dashboard.  
2. Super Admin views the list of all past broadcasts with key metadata.  
3. Super Admin clicks on a broadcast to view its full details and analytics.  
4. Super Admin clicks 'Duplicate' to create a new broadcast pre-filled with the same content and settings.  
5. Super Admin edits as needed and sends or schedules the duplicate.  
6. Alternatively, Super Admin clicks 'Resend' to dispatch the same broadcast to the same or a new audience.

#### **Acceptance Criteria**

| \# | Criteria |
| ----- | ----- |
| AC01 | Broadcast History displays all broadcasts with the following metadata: Broadcast Name, Date Sent, Sender, Recipient Count, Open Rate, and Click Rate. |
| AC02 | Clicking on any broadcast opens a detail view showing full content, audience filters used, and complete analytics. |
| AC03 | The Duplicate action creates a new broadcast pre-filled with the original's content, subject line, and sender details. Audience selection is reset and must be re-confirmed. |
| AC04 | The Resend action allows the Super Admin to dispatch the same broadcast content to the same audience or a modified audience. |
| AC05 | Broadcast History is searchable and can be sorted by date sent, broadcast name, and recipient count. |
| AC06 | Duplicate and Resend actions are logged in the Audit Trail. |

## **06 — Functional Requirements**

### **6.1 Broadcast Dashboard**

Dedicated dashboard within Platform Admin displaying a real-time summary of all broadcast activity.

**Summary Counters**

| Metric | Description |
| ----- | ----- |
| Total Broadcasts | All broadcasts ever created |
| Draft Broadcasts | Saved but not yet sent or scheduled |
| Scheduled Broadcasts | Queued for future delivery |
| Sent Broadcasts | Successfully dispatched |
| Failed Broadcasts | Broadcasts with delivery errors |

**Performance Metrics**

| Metric | Description |
| ----- | ----- |
| Total Sent | Number of emails dispatched |
| Delivered | Confirmed deliveries |
| Opened | Unique opens recorded |
| Clicked | Unique link clicks recorded |
| Failed | Undelivered emails |

### **6.2 Create Broadcast**

**Required Fields**

* Broadcast Title (internal reference only)  
* Subject Line (visible to recipients)  
* Sender Name  
* Reply-To Email

**Email Content Editor**

Rich text editor supporting formatted text, headings, lists, embedded images, links, buttons, and tables.

**Attachments**

The Super Admin may attach PDFs, images, and general documents to a broadcast.

### **6.3 Audience Selection**

| Option | Filter Criteria |
| ----- | ----- |
| Send to Everyone | All active users on the platform |
| By User Role | Judges, Magistrates, Registrars, Clerks, Lawyers, Court Administrators, ICT Officers |
| By Court | Specific court, court type, state, or judicial division |
| By Subscription Plan | Basic, Standard, Essential, or Pro |
| Custom Selection | Manual search and individual user selection |
| Combined Filters | Multiple criteria applied simultaneously with AND logic |

### **6.4 Broadcast Preview**

* Desktop view  
* Mobile view

### **6.5 Send Options**

| Option | Behaviour |
| ----- | ----- |
| Send Immediately | Dispatches the broadcast as soon as the Super Admin confirms |
| Schedule | Super Admin selects a future date and time for automatic dispatch |
| Save as Draft | Stores the broadcast for later editing and sending |

### **6.6 Broadcast Statuses**

| Status | Meaning |
| ----- | ----- |
| Draft | Created and saved, not yet scheduled or sent |
| Scheduled | Set for future delivery at a specific date and time |
| Sending | Currently being dispatched to recipients |
| Sent | Fully delivered to all intended recipients |
| Failed | Dispatch encountered errors; delivery incomplete |
| Cancelled | Scheduled broadcast was cancelled before dispatch |

## **07 — Analytics & Tracking**

### **7.1 Delivery Metrics**

* Total recipients targeted  
* Successfully delivered count  
* Failed delivery count

### **7.2 Engagement Metrics**

* Total unique opens  
* Total unique link clicks

### **7.3 Audience Breakdown**

Opens and engagement viewable filtered by user role, court, and state.

## **08 — Broadcast Templates**

### **System Templates**

| Template Name | Use Case |
| ----- | ----- |
| Feature Release | Announcing new platform features |
| Maintenance Notice | Scheduled or emergency downtime communication |
| Subscription Reminder | Renewal reminders for courts on lapsing plans |
| Deployment Notice | Informing courts ahead of deployment activities |
| Training Invitation | Inviting users to onboarding or training sessions |
| Security Alert | Urgent security-related communications |

### **Custom Templates**

The Super Admin can create, name, save, and reuse custom templates for any recurring communication type.

## **09 — Broadcast History**

All sent broadcasts are retained in a searchable history log. The following data is captured per broadcast:

* Broadcast Name  
* Date Sent  
* Sender  
* Recipient Count  
* Open Rate  
* Click Rate

From Broadcast History, the Super Admin can:

* View the full details and analytics of any past broadcast  
* Duplicate a broadcast to use as the basis for a new one  
* Resend a previous broadcast to the same or a modified audience

## **10 — Permissions & Access**

There is a single admin role on the platform: **Super Admin**. All broadcast permissions described in this document apply exclusively to the Super Admin. There is no separate Admin or Viewer role for this feature.

The Super Admin has full access to all broadcast actions:

* Create new broadcasts  
* Edit broadcasts in any pre-send status  
* Schedule broadcasts for future delivery  
* Send broadcasts immediately  
* Delete draft broadcasts  
* Cancel scheduled broadcasts  
* View full analytics and broadcast history

## **11 — Audit Trail**

Every action within the Email Broadcast System is logged automatically. Each entry captures the Super Admin user ID, action performed, timestamp, and Broadcast ID.

| Action | Trigger |
| ----- | ----- |
| Broadcast Created | New broadcast saved (draft or immediately sent) |
| Broadcast Edited | Any update made to an existing broadcast |
| Broadcast Scheduled | Future delivery time set |
| Broadcast Sent | Dispatch confirmed and initiated |
| Broadcast Cancelled | Scheduled broadcast manually cancelled |
| Template Created | New custom template saved |
| Template Deleted | Custom template removed |

## **12 — Technical Requirements**

### **12.1 Backend APIs**

| Endpoint | Function |
| ----- | ----- |
| Create Broadcast | Saves a new broadcast as draft or initiates sending |
| Update Broadcast | Edits an existing broadcast before dispatch |
| Delete Draft | Removes a broadcast in Draft status |
| Send Broadcast | Triggers immediate dispatch to selected audience |
| Schedule Broadcast | Sets a broadcast for future delivery |
| Get Analytics | Returns delivery and engagement data per broadcast |
| Get Audience Count | Returns estimated recipient count for a given filter set |

### **12.2 Database Entities**

**Broadcast**

| Field | Description |
| ----- | ----- |
| ID | Unique identifier |
| Title | Internal broadcast name |
| Subject | Email subject line |
| Content | Email body (rich text / HTML) |
| Status | Current broadcast status |
| Created By | Super Admin user ID |
| Created At | Timestamp of creation |
| Scheduled At | Scheduled dispatch time (nullable) |
| Sent At | Actual send timestamp (nullable) |

**Broadcast Recipient**

| Field | Description |
| ----- | ----- |
| Broadcast ID | Foreign key to Broadcast |
| User ID | Foreign key to User |
| Delivery Status | Delivered / Failed |
| Opened | Boolean, true if email was opened |
| Clicked | Boolean, true if a link was clicked |

### **12.3 Frontend**

**Required Pages**

* Broadcast Dashboard  
* Create / Edit Broadcast  
* Audience Builder  
* Template Library  
* Analytics View  
* Broadcast History

**Required Screens**

* Broadcast List  
* Create Broadcast form  
* Audience Filter Builder  
* Rich Text Email Editor  
* Analytics Dashboard  
* Broadcast Detail View

## **13 — Future Enhancements (Phase 2\)**

| Enhancement | Description |
| ----- | ----- |
| SMS Broadcasts | Send SMS messages to users via mobile number |
| WhatsApp Notifications | Broadcast via WhatsApp Business API |
| In-App Notifications | Notify users within the JudicAI interface |
| Automated Campaigns | Trigger broadcasts based on user actions or platform events |
| A/B Testing | Test subject lines and content variants |
| Multi-language Support | Send broadcasts in multiple languages |
| Subscription Automation | Auto-trigger renewal reminders based on plan expiry dates |

## **14 — Success Metrics**

| Metric | Target |
| ----- | ----- |
| Broadcasts sent monthly | At least 4 broadcasts per month within 60 days of launch |
| Open rate | Above 30% average open rate within 90 days |
| Click rate | Above 10% average click rate within 90 days |
| External tool usage | Full elimination of external email tool usage post-launch |
| Feature adoption | Measurable uplift in new feature engagement following broadcast announcements |

