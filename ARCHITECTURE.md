# Architecture & System Design

## System Overview

```
┌─────────────────────────────────────────────────────────────┐
│                   MOBILE DEVICE (iOS/Android)               │
│  ┌──────────────────────────────────────────────────────┐   │
│  │     Canvas Power App - Inspection Form               │   │
│  │  • Photo capture                                     │   │
│  │  • Barcode scanning                                  │   │
│  │  • Signature pad                                     │   │
│  │  • Form validation                                   │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                            ↓ (Online/Offline)
┌─────────────────────────────────────────────────────────────┐
│              MICROSOFT POWER PLATFORM                        │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Dataverse (Data Storage & Sync)                     │   │
│  │  • Inspections Table                                 │   │
│  │  • InspectionItems Table                             │   │
│  │  • Users Table                                       │   │
│  │  • TaskAssignments Table                             │   │
│  └──────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Power Automate (Workflows & Automation)             │   │
│  │  • PDF Report Generation                             │   │
│  │  • Email Notifications                               │   │
│  │  • Data Validation & Cleanup                         │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                            ↓ (Future Phase)
┌─────────────────────────────────────────────────────────────┐
│         D365 FINANCE & OPERATIONS (ERP)                      │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Virtual Entities                                    │   │
│  │  • Device Master Data                                │   │
│  │  • Rental Contracts                                  │   │
│  │  • Equipment Tracking                                │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

## Data Flow Diagram

### Inspection Submission Flow
```
1. User fills inspection form on mobile app
   ↓
2. Form validation triggers (Power Fx)
   ↓
3. All data + attachments (photos, signature) submitted
   ↓
4. Data stored in Dataverse
   ↓
5. Power Automate triggered (on row create/update)
   ↓
6. PDF generated from inspection data
   ↓
7. PDF stored in Dataverse & email sent to office
   ↓
8. Inspection record marked complete
```

### Offline Synchronization
```
App Online State
├─ Real-time sync to Dataverse
├─ Photos uploaded immediately
└─ UI reflects server state

App Offline State
├─ Form data cached locally
├─ Photos stored in app storage
├─ Signature temporarily stored
└─ Queue submission when online

Sync on Reconnect
├─ Upload queued inspections
├─ Download latest task assignments
├─ Refresh device master data
└─ Update user UI with sync status
```

## Dataverse Schema

### Inspections Table
```
Column Name          | Type           | Purpose
─────────────────────────────────────────────────────
InspectionID         | GUID (Primary) | Unique identifier
DeviceSerialNumber   | Text           | Device barcode/serial
DeviceName           | Text           | Device model/type
InspectionDate       | DateTime       | When inspection occurred
InspectorName        | Lookup         | Link to Users table
InspectionStatus     | OptionSet      | Draft/Submitted/Completed
DeviceCondition      | OptionSet      | Good/Fair/Poor/Damaged
OverallNotes         | Text           | General comments
PDFReportURL         | Hyperlink      | Link to generated PDF
CreatedOn            | DateTime       | System timestamp
ModifiedOn           | DateTime       | Last update
```

### InspectionItems Table
```
Column Name          | Type           | Purpose
─────────────────────────────────────────────────────
ItemID               | GUID (Primary) | Unique identifier
InspectionID         | Lookup         | Parent inspection
ItemCategory         | OptionSet      | Screen/Battery/Housing/Other
ItemCondition        | OptionSet      | OK/Minor Damage/Major Damage
PhotosJSON           | Text           | JSON array of photo metadata
Notes                | Text           | Item-specific notes
Timestamp            | DateTime       | When item recorded
```

### Users Table
```
Column Name          | Type           | Purpose
─────────────────────────────────────────────────────
UserID               | GUID (Primary) | Unique identifier
Name                 | Text           | Staff member name
Email                | Email          | Contact email
Department           | OptionSet      | Office/Field/Management
IsActive             | Boolean        | Active status
CreatedOn            | DateTime       | Record creation
```

### TaskAssignments Table
```
Column Name          | Type           | Purpose
─────────────────────────────────────────────────────
TaskID               | GUID (Primary) | Unique identifier
AssignedToUserID     | Lookup         | User table link
DeviceID             | Text           | Device to inspect
Status               | OptionSet      | Pending/InProgress/Completed
Priority             | OptionSet      | Low/Medium/High
DueDate              | DateTime       | Deadline
CreatedOn            | DateTime       | Task creation date
```

## Canvas App Architecture

### Screen Structure
```
App
├── Home Screen
│   ├── Welcome banner
│   ├── Today's tasks list
│   ├── Quick stats (inspections completed)
│   └─ Navigation menu
│
├── Inspection List Screen
│   ├── Filter controls (By Date/Status/Inspector)
│   ├── Sortable gallery
│   └─ Create new inspection button
│
├── New/Edit Inspection Screen
│   ├── Device Information Section
│   │   ├─ Serial Number / Barcode input
│   │   ├─ Device Name dropdown
│   │   └─ Inspection Date picker
│   │
│   ├── Condition Assessment Section
│   │   ├─ Photo capture controls
│   │   ├─ Damage checkboxes
│   │   ├─ Condition rating dropdowns
│   │   └─ Notes text box
│   │
│   ├── Signature Section
│   │   ├─ Signature pad
│   │   ├─ Inspector name display
│   │   └─ Time stamp
│   │
│   └── Submit Section
│       ├─ Validation error display
│       ├─ Submit button
│       └─ Save draft button
│
├── Task Details Screen
│   ├─ Task information
│   ├─ Device details
│   ├─ Assigned date/due date
│   └─ Action buttons (Start/Complete)
│
└── Completed Inspections Screen
    ├─ Inspection gallery
    ├─ PDF preview
    └─ Export options
```

## Data Storage Strategy

### Dataverse Storage
- **Tables:** Normalized Dataverse tables as defined above
- **Attachments:** Photos stored as Dataverse file columns
- **Signatures:** Base64 encoded in text columns
- **PDFs:** Generated and stored via Power Automate

### File Storage
- **Photos:** Dataverse file column (10MB limit per file)
- **Signature Images:** Dataverse blob storage or SharePoint
- **Generated PDFs:** SharePoint document library or Dataverse

### Caching Strategy
- User/device master data cached on app startup
- Task assignments refreshed on screen load
- Inspection data cached per session
- Clear cache on logout

## Authentication & Security

- **Authentication:** Azure AD (via Power Platform)
- **Row-Level Security (RLS):** Filter inspections by assigned user
- **Data Encryption:** Dataverse encryption at rest
- **Transport Security:** TLS 1.2+ for all data transfers
- **Session Management:** Automatic timeout after 30 minutes inactivity

## Performance Considerations

### Mobile Optimization
- Lazy load images
- Compress photos before upload (target: <500KB per image)
- Pagination for inspection lists (50 records per page)
- Limit dropdown options (max 1000 items)

### Offline Capability
- Store up to 10 pending inspections locally
- Queue-based submission on reconnection
- Conflict resolution: server data wins
- User notification of sync status

### Scalability
- **50 concurrent users** can handle 500+ daily inspections
- Dataverse handles historical data archiving
- Implement data retention policies after 2 years

## Integration Points

### Phase 1: Current (Dataverse ↔ Canvas App)
- Direct Dataverse connectors in Canvas App
- Power Automate for PDF generation
- Built-in email notifications

### Phase 2: PDF Generation (Power Automate)
- Cloud flow triggered on inspection submission
- HTML template rendered with inspection data
- PDF generated and stored
- Email sent with PDF attachment

### Phase 3: D365FO Integration (Future)
- Virtual entities for device master data
- Real-time device lookup
- Sync inspection data to ERP
- Integration via Dataverse → D365FO connectors

### Phase 4: Advanced Analytics (Future)
- Power BI dashboards for inspection trends
- Anomaly detection for damaged devices
- Maintenance scheduling insights

## Error Handling & Resilience

### Network Failures
- Automatic offline detection
- Queue inspection submissions
- Retry with exponential backoff
- User notifications

### Validation Failures
- Real-time field validation (Power Fx)
- Show validation errors before submission
- Highlight required fields
- Clear error messages

### Data Corruption
- Dataverse automated backups
- Audit trail for all changes
- Rollback capability via Power Automate

---

**Related Documentation:**
- [BEST_PRACTICES.md](./BEST_PRACTICES.md) - Development guidelines
- [Dataverse Schema Details](./docs/dataverse-schema.md)
- [Power Automate Workflows](./docs/power-automate-workflows.md)
