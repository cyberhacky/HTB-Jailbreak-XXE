# Hack The Box - Jailbreak (Very Easy)

## Overview

**Jailbreak** is a beginner-friendly Hack The Box web challenge focused on identifying and exploiting an XML External Entity (XXE) vulnerability.

The application simulates a Pip-Boy firmware update interface where users can submit XML configuration files to initiate a firmware upgrade. While the interface appears straightforward, accepting raw XML input immediately raises questions about how the backend processes user-supplied data.

The objective of this assessment was to analyze the application's functionality, identify any weaknesses in the XML processing logic, and determine whether sensitive files could be accessed through improper parser configuration.

Throughout this write-up, I'll walk through the methodology I used—from initial reconnaissance and source code analysis to vulnerability validation and successful exploitation—while also discussing why the vulnerability exists and how it can be mitigated.
