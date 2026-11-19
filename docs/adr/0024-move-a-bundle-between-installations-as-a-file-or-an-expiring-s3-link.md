---
status: accepted
date: 2026-11-20
decided-by: Niko4417
ticket: https://github.com/Niko4417/OPT-cc/issues/21
---

# Move a bundle between installations as a file or an expiring S3 link

V1 has no shared server (ADR-0015), so two installations have no channel between them. Export writes the project memory bundle to the installation's own S3 bucket in Frankfurt and shows a download link that expires. Import accepts either an uploaded file or such a link. A lifecycle rule deletes exported files from the bucket after 7 days. How the consultant sends the link or the file is their choice, because the file is encrypted (ADR-0023).

## Considered options

- A channel between installations, or a shared registry. Rejected because V1 has no shared service (ADR-0015).
- Download and upload only. Rejected because a bundle with recordings can reach gigabytes, which is slow to route through a laptop. With a link, the file goes from S3 to the importing installation directly.

## Consequences

- The spec sets the lifetime of the link. A presigned S3 URL that was signed with temporary role credentials stops working when those credentials expire, so the lifetime depends on how the installation signs it.
- The 7-day rule removes only the exporter's file in S3. Deleting content from copies that were already imported stays open on the map.
