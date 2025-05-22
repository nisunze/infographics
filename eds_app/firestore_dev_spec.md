# Firestore Application Specification - Interactive Infographic

## Application Architecture Highlights
This specification details the backend data structure for a robust, offline-first mobile application.

- **Flutter Application**: Cross-platform mobile app for seamless user experience.

- **Offline-First Approach**: Ensures functionality even without an active internet connection.

- **Hive & Firestore Sync**: Local data storage with Hive, syncing to Firestore. Authentication is handled directly (e.g., Firebase Auth).

1. Overview of Firestore Backend
This document outlines the Firestore data structure, schema, and access control mechanisms. The design emphasizes consistent metadata, robust location data handling, project-based organization, and role-based access control for the backend supporting the Flutter application.

2. Collection Naming Convention
All form-specific collections follow a standardized naming convention for clarity and project association.

Format: <eds_project_id>_<form_name>

Examples: sffebflw_burera_notes, sffebflw_burera_customers

3. Document Schema for Form Records
Every document includes a mandatory metadata object.

3.1. Universal Metadata Fields
Key categories of fields present in every record's metadata.
(The infographic displays a Donut chart illustrating these categories: Timestamps (2 fields), Location (1 field), User Info (3 fields), Flags/IDs (2 fields)).

Schema:

{
  "metadata": {
    "created_at": "<Timestamp>",
    "updated_at": "<Timestamp>",
    "location": "<GeoPoint>",
    "is_deleted": "<boolean>",
    "user_email": "<string>",
    "user_id": "<string>",
    "project_id": "<string>"
  }
  // ... other form-specific fields
}

3.2. Location Data Handling
Universal metadata.location (GeoPoint)
A Firestore GeoPoint ({latitude, longitude}) is mandatory in the metadata object of ALL form records.
For forms not specifically capturing location, this might be a default project location or general entry. For dedicated location-capturing forms, this GeoPoint is derived from the detailed capture (see below), ensuring a consistent, indexable point for all records. Used for quick map centering and general queries.

Dedicated Location-Capturing Forms: Dual Storage & Detailed Model
Forms specifically designed to capture precise geographic information will store a detailed location object (e.g., detailed_location_capture) at the root level of the document IN ADDITION TO the derived metadata.location GeoPoint. This means location data for these forms is effectively stored twice to serve different purposes.

Example detailed_location_capture object structure:

{
  // ... other form-specific fields
  "metadata": { /* ... as defined in 3.1 ... */ },
  "detailed_location_capture": {
    "latitude": -1.9638561,       // <number>
    "longitude": 30.0313834,      // <number>
    "accuracy": 20.1,             // <number> GPS accuracy in meters
    "altitude": 840.1,            // <number> Elevation in meters
    "heading": 0,                 // <number> Direction (0-359.9 degrees), optional
    "speed": 0,                   // <number> Speed in m/s, optional
    "speedAccuracy": 1.5,         // <number> Speed accuracy in m/s, optional
    "location_timestamp": "<Timestamp>" // Firestore Timestamp of capture
  }
}

Location Accuracy Threshold (for dedicated forms): 10m (MAX_ACCEPTABLE_ACCURACY)

Source: All location data is intended to come from a geolocator library.

3.3. Soft Deletion
Records are hidden by setting metadata.is_deleted = true. Frontend filters these out from normal views.

3.4. Example Base Form: `sample_form`
This `sample_form` serves as a foundational template for creating other specific data collection forms. It incorporates the standard `metadata` and, being a location-aware form, the `detailed_location_capture` object. Other forms can extend this base by adding their unique fields.

**Core Principle:** The consistent use of `metadata` and `detailed_location_capture` (where applicable) across all forms promotes a strong core structure, reducing redundancy and ensuring data integrity. This allows for a common library/logic in the Flutter app to handle these base fields.

Schema for a document in `eds_project_id_sample_form` collection:

```json
{
  "metadata": {
    "created_at": "<Timestamp>",
    "updated_at": "<Timestamp>",
    "location": "<GeoPoint>", // Derived from detailed_location_capture.latitude/longitude
    "is_deleted": false,
    "user_email": "<string>",
    "user_id": "<string>",
    "project_id": "<string>" // Matches eds_project_id prefix
  },
  "detailed_location_capture": { // As this is a location-taking form
    "latitude": "<number>",
    "longitude": "<number>",
    "accuracy": "<number>",
    "altitude": "<number>",
    "heading": "<number>",
    "speed": "<number>",
    "speedAccuracy": "<number>",
    "location_timestamp": "<Timestamp>"
  },
  "name": "<string>",                 // Name of the sample/item
  "notes": "<string>",                // General notes
  "image": "<string>",                // URL to a primary image
  "images": [                       // List of URLs for additional images
    "<string_url_1>",
    "<string_url_2>"
    // Max items defined by MAX_SAMPLE_FORM_IMAGES constant
  ]
  // Potentially other sample_form specific fields
}
```

A constant like `MAX_SAMPLE_FORM_IMAGES` (e.g., 5) would be defined in the application to limit the number of image URLs in the `images` list.

4. users Collection Schema
Stores user profiles and project affiliations. Path: users/{user_id}

{
  "email": "<string>",
  "name": "<string>",
  "last_login": "<Timestamp>",
  "active_project_id": "<eds_project_id>",
  "active_project_updated_at": "<Timestamp>",
  "active_customer_collection": "<string>",
  "active_infrastructure_collection": "<string>",
  "projects": { // Queried for efficient project access
    "owner": ["<eds_project_id>", ...],
    "project_survey_managers": ["<eds_project_id>", ...],
    "project_field_surveyors": ["<eds_project_id>", ...]
  }
}

Efficiency Note: To get projects a user has access to, the application queries the projects map within this user document, avoiding inefficient scans of all eds_project documents.

5. eds_project Collection Schema
Stores project metadata and role assignments by user email. Path: eds_project/{eds_project_id}

Critical Backend Control:
eds_project documents are exclusively created and deleted by a backend API.
The frontend application cannot create, delete, or hide these project documents.
Frontend actions are limited to viewing project details and, for authorized roles (Owner), assigning or de-assigning users to project_survey_manager and project_field_surveyor roles within an existing project.

{
  "eds_project_id": "<string>",
  "project_name": "<string>",
  "project_location": "<string>",
  "owner": "<user_email>", // Unique Owner
  "project_survey_managers": ["<user_email>", ...],
  "project_field_surveyors": ["<user_email>", ...],
  "project_editors": ["<user_email>", ...],
  "project_viewers": ["<user_email>", ...],
  "creation_time": "<Timestamp>",
  "updated_at": "<Timestamp>",
  "variables": "<object>"
}

(The infographic displays a Horizontal Bar Chart for Illustrative Role Counts: Owner (1), Survey Managers (2), Field Surveyors (10), Editors (1), Viewers (3)).

6. Access Control Logic (Frontend Perspective)
Permissions for frontend actions related to projects and their data. Project document lifecycle (create/delete) is backend-controlled.

**Owner:**
- Full CRUD on form data within the project.
- Can assign/de-assign users to `project_survey_manager` & `project_field_surveyor` roles for the project.
- Views project details.
- Cannot create/delete/hide project document or transfer ownership via frontend.

**Project Survey Manager:**
- Assigned by Owner.
- Manages (adds/removes) `project_field_surveyors` for the project.
- Read all project form records.
- Create/Update of form records may be allowed.
- Views project details.

**Project Field Surveyor:**
- Assigned by Owner/Manager.
- Create form records.
- R/U own form records.
- Can add (but not remove) other users to the `project_field_surveyors` role for the project.
- Views project details.

Backend roles (Viewer, Editor) for admin tasks, enforced by Security Rules, have broader direct DB access.

7. Firestore Indexing Strategy
Efficient querying is supported by a combination of automatic and managed indexes.

Standard Indexes (Backend Created)
The backend service automatically creates standard composite indexes during project creation for optimal performance on common queries. These typically include indexes on:

- `metadata.updated_at` (for sorting by update time)
- `metadata.user_email` (for filtering by user)
- `metadata.is_deleted` (for filtering soft-deleted records)

Custom/Additional Indexes
If the Flutter application developers identify a need for additional Firestore indexes:

1. Developers can create these indexes in their development/testing Firebase project via the console or CLI.
2. Once validated, these new index definitions must be communicated to the backend team.
3. The backend team will then incorporate these additional index configurations into the standard project creation process, ensuring they are available for all future projects.

8. General Rules
- Frontend queries filter out soft-deleted records (metadata.is_deleted == true) by default.
- Firestore Security Rules are critical for database-level permission enforcement, aligning with this defined logic.
- Users operate within their assigned projects and roles.

---

*Firestore Application Specification Infographic © 2025*