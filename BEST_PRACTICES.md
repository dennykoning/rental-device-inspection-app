# Canvas Power App Best Practices - Inspection App

## 1. Form Design & UX

### Layout Principles
✅ **DO:**
- Use single-column layout on mobile
- Group related fields (Device Info → Condition → Signature)
- Show progress indicators (1/4 sections complete)
- Use maximum 3-4 fields per screen section
- Provide clear section headers with icons

❌ **DON'T:**
- Force horizontal scrolling
- Put more than 5 fields on one section
- Use small touch targets (<48px)
- Display all optional fields at once

### Mobile-First Component Sizes
```
Button Height:      48-56px
Input Field:        44-48px height
Padding:            12-16px
Column Gap:         8-12px
Section Margin:     12-20px
```

### Example Form Section Structure
```
┌─────────────────────────────────┐
│ 📱 DEVICE INFORMATION           │ (Header with icon)
├─────────────────────────────────┤
│ Serial Number / Barcode:        │
│ ┌──────────────────────────────┐│
│ │ [INPUT] [SCAN] [CLEAR]      ││
│ └──────────────────────────────┘│
│                                 │
│ Device Name:                    │
│ ┌──────────────────────────────┐│
│ │ [DROPDOWN - 44px height]     ││
│ └──────────────────────────────┘│
│                                 │
│ Inspection Date:                │
│ ┌──────────────────────────────┐│
│ │ [DATE PICKER - 44px]         ││
│ └──────────────────────────────┘│
└─────────────────────────────────┘
```

## 2. Input Validation

### Real-Time Validation
```powerapps
// Serial Number - At least 6 characters
SerialNumberInput.ValidationState = If(
    Len(SerialNumberInput.Value) >= 6,
    ValidationState.Valid,
    If(Len(SerialNumberInput.Value) = 0, 
        ValidationState.None, 
        ValidationState.Error
    )
)

// Error message display
SerialNumberError.Visible = SerialNumberInput.ValidationState = ValidationState.Error
SerialNumberError.Text = If(
    Len(SerialNumberInput.Value) < 6,
    "Serial number must be at least 6 characters",
    ""
)
```

### Required Field Validation
```powerapps
// Before submit - Check all required fields
SubmitButton.DisplayMode = If(
    And(
        Len(SerialNumberInput.Value) >= 6,
        !IsBlank(DeviceNameDropdown.Value),
        !IsBlank(InspectionDatePicker.Value),
        SignaturePad.Value <> Blank(),
        ConditionRating.Value <> Blank()
    ),
    DisplayMode.Edit,
    DisplayMode.Disabled
)

// Show validation summary
ValidatorText.Text = If(
    SubmitButton.DisplayMode = DisplayMode.Disabled,
    Concatenate(
        If(Len(SerialNumberInput.Value) < 6, "✗ Serial number required", ""),
        If(IsBlank(DeviceNameDropdown.Value), "✗ Device name required", ""),
        If(IsBlank(InspectionDatePicker.Value), "✗ Date required", "")
    ),
    ""
)
```

### Phone Number / Serial Number Formatting
```powerapps
// Automatic barcode cleaning (remove spaces/dashes)
TextClean(SerialNumberInput.Value)

// Format to uppercase
Upper(SerialNumberInput.Value)

// Duplicate check against existing inspections
SerialNumberInput.ValidationState = If(
    CountRows(Filter(Inspections, DeviceSerialNumber = SerialNumberInput.Value)) > 0,
    ValidationState.Warning, // Already inspected
    ValidationState.Valid
)
```

## 3. Image & Photo Handling

### Photo Capture Best Practice
```powerapps
// Camera control setup
CameraControl1.Photo = Photo

// Convert to compressed image on capture
AddColumn(
    [Photo],
    ImageData,
    ImageAddColumns(Photo, {
        // Resize to 1024px max width
        // Compress to 70% quality JPEG
    })
)

// Store in collection first (before submission)
ClearCollect(
    colPhotos,
    {
        PhotoID: GUID(),
        PhotoData: CameraControl1.Photo,
        Timestamp: Now(),
        Category: "Damage Assessment"
    }
)
```

### Photo Upload to Dataverse
```powerapps
// After inspection submitted, upload photos
ForAll(
    colPhotos,
    AddAttachment(
        Inspections,
        InspectionID,
        PhotoID & ".jpg",
        ImageData
    )
)
```

### Image Gallery Display
```powerapps
// Show uploaded photos in inspection
Gallery1.Items = Filter(
    Inspections,
    InspectionID = ThisInspection.InspectionID
)
```

## 4. Barcode Scanning

### Barcode Scanner Integration
```powerapps
// Option 1: Use Barcode Scanner Control (built-in)
BarcodeScannerControl1.OnScan = Set(
    varScannedValue,
    BarcodeScannerControl1.Value
);
Set(varShowScanner, false)

// Update serial number field with scanned value
SerialNumberInput.Value = varScannedValue
```

### Barcode Result Handling
```powerapps
// Check if barcode exists in device master
Set(varDeviceExists, CountRows(Filter(Devices, SerialNumber = varScannedValue)) > 0)

If(varDeviceExists,
    Notify("Device found!", NotificationType.Success),
    Notify("Device not in system - manual entry required", NotificationType.Warning)
)

// Auto-populate device name if found
DeviceNameDropdown.Value = LookUp(
    Devices,
    SerialNumber = varScannedValue,
    DeviceName
)
```

## 5. Signature Capture

### Signature Pad Setup
```powerapps
// Signature Pad control configuration
SignaturePad1.Width = Parent.Width - 16
SignaturePad1.Height = 180
SignaturePad1.BorderColor = ColorValue("#E0E0E0")

// Clear signature button
ClearSignatureButton.OnSelect = SignaturePad1.Clear()

// Verify signature before submit
If(
    IsBlank(SignaturePad1.Value),
    Notify("Signature required", NotificationType.Error),
    Set(varSignatureData, SignaturePad1.Value)
)
```

### Store Signature in Dataverse
```powerapps
// Convert signature to base64 before storing
Set(varSignatureBase64, Base64Encode(SignaturePad1.Value))

// Submit to Dataverse
Patch(
    Inspections,
    InspectionRecord,
    {
        SignatureImage: varSignatureBase64,
        SignedBy: User().FullName,
        SignatureTimestamp: Now()
    }
)
```

## 6. Dropdown & Choice Fields

### Load Master Data Efficiently
```powerapps
// At app startup - load device list once
ClearCollect(
    colDevices,
    Filter(Devices, IsActive = true)
)

// Use collection in dropdowns (faster than live query)
DeviceNameDropdown.Items = colDevices
DeviceNameDropdown.DisplayFields = ["DeviceName"]
DeviceNameDropdown.SearchFields = ["SerialNumber", "DeviceName"]
```

### Conditional Dropdowns
```powerapps
// Condition rating depends on device type
ConditionOptions.Items = If(
    DeviceNameDropdown.Value.Type = "Mobile Phone",
    ["Screen OK", "Screen Cracked", "Battery Issue", "Not Working"],
    If(DeviceNameDropdown.Value.Type = "Tablet",
        ["Perfect", "Minor Marks", "Dent/Crack", "Inoperable"],
        ["Good", "Fair", "Poor"]
    )
)
```

### Search & Filter Dropdowns
```powerapps
// Allow search in dropdown (make it searchable)
DeviceNameDropdown.SearchFields = ["DeviceName", "SerialNumber"]

// Filter devices by category if needed
ConditionDropdown.Items = Filter(
    colDevices,
    Category = varSelectedCategory.Value
)
```

## 7. Form State Management

### Save Draft Functionality
```powerapps
// Collections to store form data
Set(varFormData, {
    SerialNumber: SerialNumberInput.Value,
    DeviceName: DeviceNameDropdown.Value,
    InspectionDate: InspectionDatePicker.Value,
    OverallCondition: ConditionRating.Value,
    Notes: NotesInput.Value,
    Photos: colPhotos,
    Signature: SignaturePad1.Value
})

// Save draft button
SaveDraftButton.OnSelect = Patch(
    Inspections,
    Defaults(Inspections),
    Extend(varFormData, {
        InspectionStatus: "Draft"
    })
);
Notify("Draft saved", NotificationType.Success)
```

### Discard Changes Confirmation
```powerapps
// Confirm before leaving without saving
If(
    Or(
        Len(SerialNumberInput.Value) > 0,
        !IsBlank(DeviceNameDropdown.Value)
    ),
    Confirm("Discard unsaved changes?", ConfirmResult.OK),
    true
)
```

## 8. Error Handling & User Feedback

### Submission Error Handling
```powerapps
// Try-catch pattern with notification
SubmitButton.OnSelect = IfError(
    Patch(
        Inspections,
        InspectionRecord,
        {
            DeviceSerialNumber: SerialNumberInput.Value,
            DeviceName: DeviceNameDropdown.Value.DeviceName,
            InspectionDate: InspectionDatePicker.Value,
            DeviceCondition: ConditionRating.Value,
            OverallNotes: NotesInput.Value,
            InspectionStatus: "Submitted"
        }
    ),
    (
        Notify("Error saving inspection: " & FirstError.Message, NotificationType.Error),
        false
    ),
    (
        Notify("Inspection submitted successfully!", NotificationType.Success),
        Navigate(CompletedScreen, ScreenTransition.Fade),
        true
    )
)
```

### Network Error Handling
```powerapps
// Detect connection issues
If(
    IsBlank(Inspections),
    Notify("No connection - inspection will sync when online", NotificationType.Warning),
    Notify("Connected to server", NotificationType.Information)
)

// Queue submission for retry
ClearCollect(
    colPendingSubmissions,
    {
        InspectionData: varFormData,
        Timestamp: Now(),
        RetryCount: 0
    }
)
```

## 9. Performance Optimization

### Lazy Loading
```powerapps
// Only load photos when user clicks on inspection
PhotoGallery.Items = If(
    !varPhotosLoaded,
    [],
    Filter(Attachments, ParentID = InspectionID)
)

// Load on demand
ShowPhotosButton.OnSelect = (
    Set(varPhotosLoaded, true),
    Set(varLoadingPhotos, true),
    ClearCollect(colInspectionPhotos, Filter(Attachments, ParentID = InspectionID)),
    Set(varLoadingPhotos, false)
)
```

### Minimize Network Calls
```powerapps
// Batch data loads
ClearCollect(
    colInspectionData,
    Inspections,
    Users,
    TaskAssignments
)

// Use collections instead of multiple data sources
Gallery1.Items = colInspectionData
```

### Delegation & Large Datasets
```powerapps
// Use Top N for pagination instead of loading all
ClearCollect(
    colInspections,
    Top(
        Filter(Inspections, InspectionStatus = "Completed"),
        50
    )
)

// Load next page on scroll
LoadMoreButton.OnSelect = ClearCollect(
    colInspections,
    Concatenate(colInspections, Top(
        Filter(Inspections, InspectionStatus = "Completed"),
        50,
        50
    ))
)
```

## 10. Accessibility & Inclusive Design

### Screen Reader Support
```powerapps
// Label all controls clearly
SerialNumberInput.AccessibleLabel = "Device Serial Number Input"
BarcodeScanButton.Tooltip = "Tap to scan device barcode"

// Use proper heading structure
SectionHeader.Role = Heading(3)
```

### Color Contrast
- Minimum WCAG AA contrast ratio: 4.5:1
- Avoid red/green only distinction for status
- Use icons + color for status indicators

### Touch-Friendly
- Minimum 48px touch targets
- 12px minimum padding between interactive elements
- Clear visual feedback on selection

---

## Quick Reference Checklist

**Before Submitting Inspection:**
- [ ] Serial number is 6+ characters
- [ ] Device name selected
- [ ] Inspection date set
- [ ] At least one photo captured
- [ ] Condition assessment completed
- [ ] Signature provided
- [ ] Notes for any issues
- [ ] Form validated without errors

**Before Deploying to Production:**
- [ ] Tested on iOS and Android
- [ ] Tested in offline mode
- [ ] All error messages user-friendly
- [ ] Photos compress correctly
- [ ] Dataverse connections working
- [ ] Power Automate flows tested
- [ ] Performance tested with 50 users
- [ ] Accessibility reviewed

---

**Related Documentation:**
- [Formula Examples](./examples/formula-examples.md)
- [Styling Guide](./examples/styling-guide.md)
- [Dataverse Schema](./docs/dataverse-schema.md)
