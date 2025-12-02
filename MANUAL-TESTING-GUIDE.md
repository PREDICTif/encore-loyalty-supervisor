# Manual Testing Guide

## Encore Loyalty AI Feedback System

**Document Version**: 1.0  
**Created**: 2025-11-29  
**Estimated Time**: 8 hours (1 day)  
**Testers Required**: 1-2 people

---

## 📋 Table of Contents

1. [Prerequisites & Setup](#1-prerequisites--setup)
2. [Test Credentials](#2-test-credentials)
3. [Testing Schedule](#3-testing-schedule)
4. [Test Cases](#4-test-cases)
   - [A. Authentication Tests](#a-authentication-tests-45-min)
   - [B. Global Admin Dashboard Tests](#b-global-admin-dashboard-tests-45-min)
   - [C. Global Admin Analytics Tests](#c-global-admin-analytics-tests-30-min)
   - [D. Global Admin Settings Tests](#d-global-admin-settings-tests-45-min)
   - [E. User Management Tests](#e-user-management-tests-60-min)
   - [F. AI Feedback Processing Tests](#f-ai-feedback-processing-tests-90-min)
   - [G. Venue Impersonation Tests](#g-venue-impersonation-tests-30-min)
   - [H. Venue Admin Dashboard Tests](#h-venue-admin-dashboard-tests-45-min)
   - [I. Venue Admin Settings Tests](#i-venue-admin-settings-tests-30-min)
   - [J. Venue Admin Analytics Tests](#j-venue-admin-analytics-tests-30-min)
   - [K. Email Functionality Tests](#k-email-functionality-tests-45-min)
   - [L. Error Handling Tests](#l-error-handling-tests-30-min)
5. [Bug Report Template](#5-bug-report-template)
6. [Test Results Summary](#6-test-results-summary)

---

## 1. Prerequisites & Setup

### Access Requirements
- [ ] Browser: Chrome, Firefox, or Edge (latest version)
- [ ] Network access to the application servers
- [ ] Test credentials (provided below)

### Application URLs
```
Frontend: http://34.192.147.181:3000
Backend API: http://34.192.147.181:8000
API Documentation: http://34.192.147.181:8000/docs
```

> **Note**: These URLs use the EC2 public IP address. Ensure ports 3000 and 8000 are open in the EC2 security group.

### Before Starting
1. Clear browser cache and cookies
2. Open browser developer tools (F12) → Console tab
3. Keep a notes document open for recording issues
4. Take screenshots of any unexpected behavior

---

## 2. Test Credentials

### Global Admin Account (Primary)
```
Email: tjrottet@encoreloyalty.net
Password: password222
Role: Global Admin
Access: Full system access
```

### Global Admin Account (Secondary)
```
Email: admin@encore.com
Password: password123
Role: Global Admin
Access: Full system access
```

### Venue Admin Account
```
Email: venuemanager@opus.com
Password: venue123
Role: Venue Admin
Venue: Opus Ocean Grille (ID: 57)
Access: Venue-scoped only
```

---

## 3. Testing Schedule

| Time Block | Section | Duration |
|------------|---------|----------|
| 9:00 - 9:45 | A. Authentication | 45 min |
| 9:45 - 10:30 | B. Global Admin Dashboard | 45 min |
| 10:30 - 11:00 | C. Global Admin Analytics | 30 min |
| 11:00 - 11:45 | D. Global Admin Settings | 45 min |
| 11:45 - 12:45 | E. User Management | 60 min |
| **12:45 - 13:30** | **LUNCH BREAK** | **45 min** |
| 13:30 - 15:00 | F. AI Feedback Processing | 90 min |
| 15:00 - 15:30 | G. Venue Impersonation | 30 min |
| 15:30 - 16:15 | H. Venue Admin Dashboard | 45 min |
| 16:15 - 16:45 | I. Venue Admin Settings | 30 min |
| 16:45 - 17:15 | J. Venue Admin Analytics | 30 min |
| 17:15 - 18:00 | K. Email & L. Error Handling | 45 min |

---

## 4. Test Cases

---

### A. Authentication Tests (45 min)

#### A1. Valid Login - Global Admin
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Navigate to http://34.192.147.181:3000 | Login form displayed |
| 2 | Enter email: `tjrottet@encoreloyalty.net` | Email field accepts input |
| 3 | Enter password: `password222` | Password field shows dots |
| 4 | Click "Sign In" button | Loading indicator appears |
| 5 | Wait for redirect | Redirected to Admin Dashboard |
| 6 | Check top-right corner | User name/email displayed |
| 7 | Check sidebar menu | Admin menu items visible |

**Result**: [ ] Pass [ ] Fail  
**Notes**: _______________________________________________

#### A2. Valid Login - Venue Admin
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Logout if logged in | Redirected to login page |
| 2 | Enter email: `venuemanager@opus.com` | Email field accepts input |
| 3 | Enter password: `venue123` | Password field shows dots |
| 4 | Click "Sign In" button | Loading indicator appears |
| 5 | Wait for redirect | Redirected to Venue Dashboard |
| 6 | Check sidebar menu | Venue-specific menu items only |
| 7 | Verify no admin menu items | No "System Monitoring", "User Management", etc. |

**Result**: [ ] Pass [ ] Fail  
**Notes**: _______________________________________________

#### A3. Invalid Login - Wrong Password
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Navigate to http://34.192.147.181:3000 | Login form displayed |
| 2 | Enter email: `tjrottet@encoreloyalty.net` | Email accepted |
| 3 | Enter password: `wrongpassword` | Password accepted |
| 4 | Click "Sign In" button | Loading indicator appears |
| 5 | Wait for response | Error message: "Invalid credentials" or similar |
| 6 | Check URL | Remains on login page |

**Result**: [ ] Pass [ ] Fail  
**Notes**: _______________________________________________

#### A4. Invalid Login - Non-existent User
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Navigate to http://34.192.147.181:3000 | Login form displayed |
| 2 | Enter email: `nonexistent@test.com` | Email accepted |
| 3 | Enter password: `anypassword` | Password accepted |
| 4 | Click "Sign In" button | Error message displayed |

**Result**: [ ] Pass [ ] Fail  
**Notes**: _______________________________________________

#### A5. Logout
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Login with valid credentials | Dashboard displayed |
| 2 | Click user menu (top-right) | Dropdown menu appears |
| 3 | Click "Logout" | Redirected to login page |
| 4 | Try to access dashboard URL directly | Redirected back to login |

**Result**: [ ] Pass [ ] Fail  
**Notes**: _______________________________________________

#### A6. Session Persistence
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Login with valid credentials | Dashboard displayed |
| 2 | Note the current page | Remember URL |
| 3 | Close browser tab (not window) | Tab closed |
| 4 | Open new tab, navigate to same URL | Still logged in (session maintained) |
| 5 | Refresh the page | Remains logged in |

**Result**: [ ] Pass [ ] Fail  
**Notes**: _______________________________________________

---

### B. Global Admin Dashboard Tests (45 min)

**Prerequisite**: Login as Global Admin (tjrottet@encoreloyalty.net)

#### B1. Dashboard Loads with Real Data
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Navigate to Dashboard (should be default) | Dashboard page loads |
| 2 | Check "Total Venues" card | Shows number (should be ~18) |
| 3 | Check "AI Enabled" card | Shows number (should be ~1) |
| 4 | Check "Total Feedback" card | Shows large number (~61,000+) |
| 5 | Check "Avg Rating" card | Shows rating (should be ~4.75) |
| 6 | Verify no "Loading..." stuck states | All cards show data |

**Result**: [ ] Pass [ ] Fail  
**Notes**: _______________________________________________

#### B2. Response Volume Chart
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Locate "Response Volume" chart | Chart is visible |
| 2 | Check chart has data | Line/bar chart with data points |
| 3 | Hover over data points | Tooltip shows values |
| 4 | Check date labels on X-axis | Readable date labels |

**Result**: [ ] Pass [ ] Fail  
**Notes**: _______________________________________________

#### B3. Quick Actions
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Locate "Quick Actions" section | Section visible |
| 2 | Click "View All Venues" (if exists) | Navigates to venues list |
| 3 | Click "View Feedback" (if exists) | Navigates to feedback page |
| 4 | Navigate back to Dashboard | Dashboard reloads correctly |

**Result**: [ ] Pass [ ] Fail  
**Notes**: _______________________________________________

#### B4. Dashboard Refresh
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Note current metric values | Record values |
| 2 | Click refresh button (if available) | Loading indicator appears |
| 3 | Wait for refresh to complete | Data reloads |
| 4 | Verify metrics still display correctly | Same or updated values |

**Result**: [ ] Pass [ ] Fail  
**Notes**: _______________________________________________

---

### C. Global Admin Analytics Tests (30 min)

**Prerequisite**: Login as Global Admin

#### C1. Global Analytics Page Loads
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Click "Analytics" in sidebar menu | Analytics page loads |
| 2 | Check for "System Analytics" header | Header visible |
| 3 | Verify page shows real data | Numbers displayed (not "Loading...") |

**Result**: [ ] Pass [ ] Fail  
**Notes**: _______________________________________________

#### C2. Venue Statistics
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Locate venue statistics section | Section visible |
| 2 | Check "Onboarded Venues" count | Shows number with percentage |
| 3 | Check "Not Onboarded" count | Shows remaining venues |
| 4 | Verify totals add up correctly | Math is correct |

**Result**: [ ] Pass [ ] Fail  
**Notes**: _______________________________________________

#### C3. Top Venues List
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Locate "Top Venues" or similar section | Section visible |
| 2 | Check venue names are displayed | Real venue names shown |
| 3 | Check feedback counts | Numbers displayed |
| 4 | Verify list is sorted (highest first) | Proper ordering |

**Result**: [ ] Pass [ ] Fail  
**Notes**: _______________________________________________

---

### D. Global Admin Settings Tests (45 min)

**Prerequisite**: Login as Global Admin

#### D1. Global Settings Page Loads
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Click "Global Settings" in sidebar | Settings page loads |
| 2 | Check for settings sections | AI Settings, Email Settings, etc. |
| 3 | Verify current values are displayed | Fields show current config |

**Result**: [ ] Pass [ ] Fail  
**Notes**: _______________________________________________

#### D2. View AI Settings
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Locate AI Settings section | Section visible |
| 2 | Check AI model display | Shows Claude model name |
| 3 | Check enabled/disabled status | Toggle or indicator visible |

**Result**: [ ] Pass [ ] Fail  
**Notes**: _______________________________________________

#### D3. View Email Settings
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Locate Email Settings section | Section visible |
| 2 | Check sender email display | Email address shown |
| 3 | Check override email (if configured) | Override email visible |

**Result**: [ ] Pass [ ] Fail  
**Notes**: _______________________________________________

#### D4. Data Synchronization Section
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Locate "Data Synchronization" section | Section visible |
| 2 | Check sync status for venues | Status indicators shown |
| 3 | Check last sync timestamp | Date/time displayed |

**Result**: [ ] Pass [ ] Fail  
**Notes**: _______________________________________________

#### D5. Trigger Manual Sync (CAREFUL)
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Locate sync button for a venue | Button visible |
| 2 | Click "Sync" button | Loading indicator appears |
| 3 | Wait for sync to complete | Success message displayed |
| 4 | Check updated sync timestamp | Timestamp updated |

**Result**: [ ] Pass [ ] Fail  
**Notes**: _______________________________________________

---

### E. User Management Tests (60 min)

**Prerequisite**: Login as Global Admin

#### E1. User Management Page Loads
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Click "User Management" in sidebar | Page loads |
| 2 | Check user list is displayed | Table with users visible |
| 3 | Check user count | Shows total users (3-4 expected) |
| 4 | Verify columns: Name, Email, Role, Status | All columns present |

**Result**: [ ] Pass [ ] Fail  
**Notes**: _______________________________________________

#### E2. View User Statistics
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Locate user statistics section | Stats cards visible |
| 2 | Check "Total Users" count | Number displayed |
| 3 | Check "Global Admins" count | Number displayed (~3) |
| 4 | Check "Venue Admins" count | Number displayed (~1) |

**Result**: [ ] Pass [ ] Fail  
**Notes**: _______________________________________________

#### E3. Create New Global Admin User
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Click "Create User" button | Modal/form opens |
| 2 | Enter Name: `Test Admin` | Field accepts input |
| 3 | Enter Email: `testadmin@test.com` | Field accepts input |
| 4 | Enter Password: `TestPass123!` | Field accepts input |
| 5 | Select Role: `Global Admin` | Role selected |
| 6 | Click "Create" or "Save" | Loading indicator |
| 7 | Wait for response | Success message, modal closes |
| 8 | Check user appears in list | New user visible |

**Result**: [ ] Pass [ ] Fail  
**Notes**: _______________________________________________

#### E4. Create New Venue Admin User
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Click "Create User" button | Modal/form opens |
| 2 | Enter Name: `Test Venue Manager` | Field accepts input |
| 3 | Enter Email: `testvenueadmin@test.com` | Field accepts input |
| 4 | Enter Password: `VenuePass123!` | Field accepts input |
| 5 | Select Role: `Venue Admin` | Role selected |
| 6 | Check for Venue ID field | Venue ID field appears |
| 7 | Enter Venue ID: `57` | Field accepts input |
| 8 | Enter Venue Name: `Test Venue` | Field accepts input |
| 9 | Click "Create" or "Save" | Success message |
| 10 | Check user appears in list | New user visible with venue info |

**Result**: [ ] Pass [ ] Fail  
**Notes**: _______________________________________________

#### E5. Test Login with Newly Created User
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Logout current session | Redirected to login |
| 2 | Login with: `testadmin@test.com` / `TestPass123!` | Login successful |
| 3 | Verify admin access | Admin dashboard visible |
| 4 | Logout | Logged out |
| 5 | Login with: `testvenueadmin@test.com` / `VenuePass123!` | Login successful |
| 6 | Verify venue access only | Venue dashboard, no admin menu |

**Result**: [ ] Pass [ ] Fail  
**Notes**: _______________________________________________

#### E6. Edit User (if available)
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Login as Global Admin | Admin access |
| 2 | Go to User Management | Page loads |
| 3 | Click Edit on a test user | Edit form opens |
| 4 | Change the name | Field updates |
| 5 | Save changes | Success message |
| 6 | Verify change persists | Updated name shown |

**Result**: [ ] Pass [ ] Fail  
**Notes**: _______________________________________________

#### E7. Delete/Deactivate User (if available)
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Find test user to delete | User in list |
| 2 | Click Delete/Deactivate button | Confirmation prompt |
| 3 | Confirm deletion | Success message |
| 4 | Verify user removed from list | User no longer visible |
| 5 | Try to login as deleted user | Login fails |

**Result**: [ ] Pass [ ] Fail  
**Notes**: _______________________________________________

---

### F. AI Feedback Processing Tests (90 min)

**Prerequisite**: Login as Global Admin

#### F1. AI Feedback Page Loads
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Click "AI Feedback" in sidebar | Page loads |
| 2 | Check feedback list appears | Table with feedback items |
| 3 | Check pagination | Page numbers or "Load More" |
| 4 | Check filters are available | Date filter, status filter |

**Result**: [ ] Pass [ ] Fail  
**Notes**: _______________________________________________

#### F2. View Feedback List
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Review feedback items in list | Items display venue name, date, rating |
| 2 | Check for customer comments | Comment text visible |
| 3 | Check for status badges | "New", "Analyzed", etc. |
| 4 | Verify rating stars/numbers | Ratings displayed correctly |

**Result**: [ ] Pass [ ] Fail  
**Notes**: _______________________________________________

#### F3. Filter by Time Period
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Locate time filter | Dropdown or tabs visible |
| 2 | Select "Last 7 Days" | List refreshes |
| 3 | Verify date range updated | Recent items only |
| 4 | Select "Last 30 Days" | List refreshes with more items |
| 5 | Select "All Time" (if available) | Shows all items |

**Result**: [ ] Pass [ ] Fail  
**Notes**: _______________________________________________

#### F4. Pagination
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Check items per page | Default count visible (e.g., 20) |
| 2 | Click "Next Page" or page 2 | New items load |
| 3 | Click "Previous Page" or page 1 | Returns to first page |
| 4 | Click last page (if visible) | Loads last page |

**Result**: [ ] Pass [ ] Fail  
**Notes**: _______________________________________________

#### F5. AI Analysis - Analyze Single Feedback
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Find a feedback item with status "New" | Item found |
| 2 | Click on the item to expand/view | Details shown |
| 3 | Click "Analyze" or "AI Analyze" button | Loading indicator |
| 4 | Wait for analysis (may take 5-15 seconds) | Analysis complete |
| 5 | Check analysis results appear | Sentiment, urgency, topics shown |
| 6 | Check status changed to "Analyzed" | Status badge updated |

**Result**: [ ] Pass [ ] Fail  
**Notes**: _______________________________________________

#### F6. AI Analysis - Review Analysis Details
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | View an analyzed feedback item | Analysis section visible |
| 2 | Check Sentiment field | "Positive", "Neutral", or "Negative" |
| 3 | Check Urgency field | "Low", "Medium", or "High" |
| 4 | Check Key Topics | List of relevant topics |
| 5 | Check Alert Required indicator | Yes/No or color indicator |

**Result**: [ ] Pass [ ] Fail  
**Notes**: _______________________________________________

#### F7. Generate AI Response Draft
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Find an analyzed feedback item | "Analyzed" status |
| 2 | Click "Generate Response" or similar | Loading indicator |
| 3 | Wait for draft generation (5-15 seconds) | Draft appears |
| 4 | Check draft content | Personalized response text |
| 5 | Check status changed to "Draft Pending" | Status updated |

**Result**: [ ] Pass [ ] Fail  
**Notes**: _______________________________________________

#### F8. Edit Draft Response
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | View a feedback item with draft | Draft visible |
| 2 | Click "Edit" or click into draft text | Edit mode enabled |
| 3 | Make changes to the text | Text is editable |
| 4 | Save changes | Success message |
| 5 | Verify changes persisted | Edited text shown |

**Result**: [ ] Pass [ ] Fail  
**Notes**: _______________________________________________

#### F9. Regenerate Draft Response
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | View a feedback item with draft | Draft visible |
| 2 | Click "Regenerate" button | Loading indicator |
| 3 | Wait for new draft | New content generated |
| 4 | Compare to previous draft | Content is different |

**Result**: [ ] Pass [ ] Fail  
**Notes**: _______________________________________________

---

### G. Venue Impersonation Tests (30 min)

**Prerequisite**: Login as Global Admin

#### G1. Access Impersonation Feature
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Look for venue selector or impersonation menu | Control visible |
| 2 | Click to open venue selection | List of venues appears |
| 3 | Verify multiple venues listed | Venue names visible |

**Result**: [ ] Pass [ ] Fail  
**Notes**: _______________________________________________

#### G2. Impersonate Venue Admin
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Select "Opus Ocean Grille" (or any venue) | Impersonation starts |
| 2 | Check UI changes to venue view | Venue-specific dashboard |
| 3 | Check sidebar shows venue menu | No admin-only items |
| 4 | Verify venue name displayed | "Opus Ocean Grille" visible |

**Result**: [ ] Pass [ ] Fail  
**Notes**: _______________________________________________

#### G3. Navigate While Impersonating
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | While impersonating, click Dashboard | Venue dashboard loads |
| 2 | Click Feedback | Venue feedback loads |
| 3 | Click Analytics | Venue analytics loads |
| 4 | Click Settings | Venue settings loads |

**Result**: [ ] Pass [ ] Fail  
**Notes**: _______________________________________________

#### G4. Exit Impersonation
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Find "Exit" or "Stop Impersonating" button | Button visible |
| 2 | Click to exit | Returns to admin view |
| 3 | Verify admin menu restored | Full admin sidebar |
| 4 | Verify admin dashboard shown | Admin dashboard data |

**Result**: [ ] Pass [ ] Fail  
**Notes**: _______________________________________________

---

### H. Venue Admin Dashboard Tests (45 min)

**Prerequisite**: Login as Venue Admin (venuemanager@opus.com)

#### H1. Venue Dashboard Loads
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | After login, check dashboard loads | Venue dashboard visible |
| 2 | Check venue name displayed | "Opus Ocean Grille" or assigned venue |
| 3 | Check metrics are venue-specific | Numbers relate to single venue |

**Result**: [ ] Pass [ ] Fail  
**Notes**: _______________________________________________

#### H2. Venue Metrics Display
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Check "Today's Feedback" card | Number displayed |
| 2 | Check "Pending Responses" card | Number displayed |
| 3 | Check "Average Rating" card | Rating displayed |
| 4 | Check "Total Feedback" card | Total count for venue |

**Result**: [ ] Pass [ ] Fail  
**Notes**: _______________________________________________

#### H3. Venue Dashboard Charts
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Locate any charts on dashboard | Charts visible |
| 2 | Hover over data points | Tooltips appear |
| 3 | Verify data relates to venue | Venue-specific data |

**Result**: [ ] Pass [ ] Fail  
**Notes**: _______________________________________________

#### H4. No Admin Access
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Check sidebar menu | No "User Management" |
| 2 | Check for "System Monitoring" | Not visible |
| 3 | Try URL: `http://34.192.147.181:3000/admin/users` directly | Access denied or redirect |
| 4 | Try URL: `http://34.192.147.181:3000/admin/system` directly | Access denied or redirect |

**Result**: [ ] Pass [ ] Fail  
**Notes**: _______________________________________________

---

### I. Venue Admin Settings Tests (30 min)

**Prerequisite**: Login as Venue Admin

#### I1. Venue Settings Page Loads
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Click "Settings" in sidebar | Settings page loads |
| 2 | Check settings are venue-specific | Venue name shown |
| 3 | Verify AI settings visible | AI toggle/configuration |

**Result**: [ ] Pass [ ] Fail  
**Notes**: _______________________________________________

#### I2. View AI Settings
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Locate AI enabled toggle | Toggle visible |
| 2 | Check current status | On or Off indicated |
| 3 | View response tone settings (if available) | Tone options visible |

**Result**: [ ] Pass [ ] Fail  
**Notes**: _______________________________________________

#### I3. Modify AI Settings (CAREFUL - May Affect Production)
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Toggle AI enabled setting | Setting changes |
| 2 | Click Save | Success message |
| 3 | Refresh page | Setting persisted |
| 4 | Toggle back to original state | Restored |

**Result**: [ ] Pass [ ] Fail  
**Notes**: _______________________________________________

---

### J. Venue Admin Analytics Tests (30 min)

**Prerequisite**: Login as Venue Admin

#### J1. Venue Analytics Page Loads
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Click "Analytics" in sidebar | Analytics page loads |
| 2 | Check data is venue-specific | Venue analytics displayed |
| 3 | Verify page title indicates venue | Venue name visible |

**Result**: [ ] Pass [ ] Fail  
**Notes**: _______________________________________________

#### J2. Rating Distribution
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Locate rating distribution section | Chart or bars visible |
| 2 | Check ratings 1-5 displayed | All rating levels shown |
| 3 | Check counts for each rating | Numbers displayed |

**Result**: [ ] Pass [ ] Fail  
**Notes**: _______________________________________________

#### J3. Status Breakdown
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Locate status breakdown section | Section visible |
| 2 | Check for "New" count | Number displayed |
| 3 | Check for "Analyzed" count | Number displayed |
| 4 | Check for "Draft" count | Number displayed |
| 5 | Check for "Sent" count | Number displayed |

**Result**: [ ] Pass [ ] Fail  
**Notes**: _______________________________________________

#### J4. Time Period Filter (if available)
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Locate time period selector | Control visible |
| 2 | Change period (7 days → 30 days) | Data updates |
| 3 | Verify metrics change | Numbers updated |

**Result**: [ ] Pass [ ] Fail  
**Notes**: _______________________________________________

---

### K. Email Functionality Tests (45 min)

**⚠️ IMPORTANT**: In development mode, all emails are redirected to `marian@dumitrascu.net`. No customer emails will actually be sent.

**Prerequisite**: Login as Global Admin

#### K1. Send Email from Draft
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Navigate to AI Feedback page | Page loads |
| 2 | Find feedback with draft response | "Draft Pending" status |
| 3 | Click "Send Email" or "Approve & Send" | Confirmation dialog |
| 4 | Confirm sending | Loading indicator |
| 5 | Wait for completion | Success message |
| 6 | Check status changed to "Email Sent" | Status updated |

**Result**: [ ] Pass [ ] Fail  
**Notes**: _______________________________________________

#### K2. Verify Email Timestamp
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | View feedback where email was sent | Item visible |
| 2 | Check for sent timestamp | Date/time displayed |
| 3 | Check for recipient info | Email address shown |

**Result**: [ ] Pass [ ] Fail  
**Notes**: _______________________________________________

#### K3. Alert Email (if applicable)
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Find negative/high-urgency feedback | Item with alert indicator |
| 2 | Analyze if not already done | Analysis shows high urgency |
| 3 | Check if manager alert sent | Alert status indicated |

**Result**: [ ] Pass [ ] Fail  
**Notes**: _______________________________________________

---

### L. Error Handling Tests (30 min)

#### L1. Network Error Simulation
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Open browser DevTools (F12) | DevTools open |
| 2 | Go to Network tab | Network tab visible |
| 3 | Enable "Offline" mode | Browser goes offline |
| 4 | Try to load a page/perform action | Error message displayed |
| 5 | Disable "Offline" mode | Back online |
| 6 | Retry action | Action succeeds |

**Result**: [ ] Pass [ ] Fail  
**Notes**: _______________________________________________

#### L2. Form Validation
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Go to Create User form | Form opens |
| 2 | Leave all fields empty | Fields empty |
| 3 | Click Submit | Validation errors shown |
| 4 | Enter invalid email format | Field accepts input |
| 5 | Click Submit | Email validation error |
| 6 | Enter very short password | Field accepts input |
| 7 | Click Submit | Password validation error |

**Result**: [ ] Pass [ ] Fail  
**Notes**: _______________________________________________

#### L3. Session Timeout (Long Test)
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Login and note the time | Time recorded |
| 2 | Wait 15+ minutes without activity | Waiting |
| 3 | Try to perform an action | May require re-login |
| 4 | Check for session timeout message | Message displayed (if applicable) |

**Result**: [ ] Pass [ ] Fail  
**Notes**: _______________________________________________

#### L4. Console Errors Check
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Open browser DevTools (F12) | DevTools open |
| 2 | Go to Console tab | Console visible |
| 3 | Navigate through all main pages | Pages load |
| 4 | Check for red errors | Document any errors |
| 5 | Check for yellow warnings | Document any warnings |

**Result**: [ ] Pass [ ] Fail  
**Notes**: _______________________________________________

---

## 5. Bug Report Template

When you find an issue, document it using this template:

```
BUG REPORT
==========

BUG-XXX: [Short Title]

Severity: [ ] Critical [ ] High [ ] Medium [ ] Low
Section: [A-L] Test Case: [#]

Steps to Reproduce:
1. 
2. 
3. 

Expected Result:
[What should happen]

Actual Result:
[What actually happened]

Screenshot: [Attach if possible]

Browser: [Chrome/Firefox/Edge] Version: [XX]
Time: [HH:MM]

Additional Notes:
[Any other relevant information]
```

---

## 6. Test Results Summary

Complete this section at the end of testing:

### Overall Results

| Section | Pass | Fail | Not Tested |
|---------|------|------|------------|
| A. Authentication | /6 | | |
| B. Dashboard | /4 | | |
| C. Analytics | /3 | | |
| D. Settings | /5 | | |
| E. User Management | /7 | | |
| F. AI Feedback | /9 | | |
| G. Impersonation | /4 | | |
| H. Venue Dashboard | /4 | | |
| I. Venue Settings | /3 | | |
| J. Venue Analytics | /4 | | |
| K. Email | /3 | | |
| L. Error Handling | /4 | | |
| **TOTAL** | **/56** | | |

### Critical Issues Found
1. 
2. 
3. 

### High Priority Issues Found
1. 
2. 
3. 

### General Observations
- 
- 
- 

### Tester Information
```
Name: ___________________________
Date: ___________________________
Start Time: _____________________
End Time: _______________________
Total Time: _____________________
```

---

**End of Manual Testing Guide**

