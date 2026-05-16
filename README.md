# ServiceHub — CCTV Service & Project Management Demo

Interactive HTML prototype for a multi-role service & project management platform built for the Kerala market (Calicut, Kollam, Ernakulam cost centers).

## Open the demo

Double-click `index.html` or visit the deployed URL.

## Personas & flows

| Persona | File | What it shows |
|---|---|---|
| **Customer** | `customer.html` | Missed call -> SMS auto-link -> service form -> ticket confirmation |
| **Technician** | `technician.html` | Login -> today's tasks -> task detail -> in-progress -> proof upload -> done |
| **Project Manager** | `pm.html` | Dashboard, projects, approvals, technicians, schedule, messages, settings |
| **Admin** | `admin.html` | Dashboard, tickets, projects, services, technicians, PMs, cost centers, reports, settings |

## Stack

- Pure HTML + CSS + vanilla JS
- Inter font, Lucide icons, Chart.js (all CDN)
- No build step, no backend

## Demo flow (suggested for client)

1. Open `index.html` -> walk through the flow diagram
2. Customer -> missed call to ticket creation
3. Admin -> open new ticket -> Convert to Service / Project -> assign technician
4. Technician -> see task -> start -> upload proof -> close
5. PM -> approve proof, view team activity
6. Admin -> cost centers comparison + reports
