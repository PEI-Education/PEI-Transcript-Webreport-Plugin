# PEI Transcript Plug-in

Adds a web report to PowerSchool SIS that allows admin users to print an official provincial transcript in English or in French.

## Accessing Reports

Make a selection of students and select *PEI Student Transcript* (under *PEI Reports*) in the Group Functions menu or on the Group Functions page.

## Constraints / Requirements

* Report must be run at the school level
* User must have a current student selection
* The report will return results for a maximum of 1500 students at a time
* Browser/available RAM may limit the number of transcripts that can be printed/saved to PDF (e.g. transcripts will load in browser, but browser may crash when trying to render for printing. 500 is recommended max batch size).
* Historical grades with a missing course numbers will print with a course number of "MISSING".

## Release History

* v2025.10.3 - Migrated data request to PowerQuery
* v2024.11 - Updated to better highlight ESAP & to include new grad reqs
* v2024.10 - Updated pagenavigation for report card changes
* v2024.6 - Updated credits table, add Audited special code, add link to PEI Report Card Data in EUI
* v2024.2.1 - Fixed current course marks displaying multiple times
* v2024.1.2 - Fixed ESAP entry in legend, added EUI nav links
* v2023.11.4 - Fixed issue with page 2 layout
* v2023.11.3 - Fixed issue with ADP showing up even when no ADP
* v2023.11.2 - Fixed issue with ADP causing transcript to not print & added ESAP blurb to page 2
* v2023.11.1 - Fixed issue where current course marks were not display
* v2023.11 - Handlebars template deployed to production (November 14, 2023). V1 files removed from repo.

> Note: changed numbering scheme to simplify versioning

* v1.9.2 - Production-ready Handlebars template with single JSON data source and updated translations (still testing)
* v1.3 - Added Essential Skills Achievement Pathway indicator to transcript legend
* v1.2 - Fixed issue with current courses
* v1.1 - Made URL link a link
* v1.1 - Updated English Programs reqs
* v1.0.1 - Respolved issue with title (role) for signature only printing on firstd page of a batch
* v1.0 - Initial release
