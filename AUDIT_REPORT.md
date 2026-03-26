Module 10 Capstone – Security Audit Report

Environment and Scope  
Host: Windows (VS Code Git Bash / MINGW64)  
Project root validated: docker-compose.yml, Dockerfile, vulnerable\_archive/  
Application: Deliberately vulnerable Django web archiving application with LLM integrations

Finding 1 — Hardcoded JWT Signing Secret (High)

Location  
vulnerable\_archive/archiver/views.py, generate\_token

Issue  
The JWT signing secret was hardcoded in the application source as SECRET = "do\_not\_share\_this". This enables token forgery if the source code is exposed and prevents safe secret rotation.

Fix Applied  
The hardcoded secret was removed from the source code.  
The JWT signing key is now read from the JWT\_SECRET environment variable.  
The application fails safely if the JWT\_SECRET variable is not set.  
The response message that disclosed the use of a hardcoded secret was removed.

Configuration Changes  
JWT\_SECRET=${JWT\_SECRET} was added to docker-compose.yml for the app-mod10 service.  
A local .env file was created to supply the signing secret at runtime.  
.env files were added to .gitignore to prevent committing secrets to version control.

Verification  
docker compose config confirmed that JWT\_SECRET is non-empty and injected into the app-mod10 container.  
docker compose up --build starts successfully and the application runs using the environment-provided secret.  
A code search confirmed no remaining references to the hardcoded JWT secret or related disclosure messages.

Regression Fix During Testing  
During testing, Django failed to start due to a missing dashboard view.  
Root cause: archiver/urls.py referenced views.dashboard, which was missing from views.py.  
Fix: Restored the dashboard view in archiver/views.py.  
Result: Django starts successfully with no system check errors and listens on <http://0.0.0.0:8000>.

Finding 2 — Stored Cross-Site Scripting (XSS) (High)

Location  
vulnerable\_archive/archiver/templates/archiver/view\_archive.html

Issue  
User-controlled content was rendered using the Django |safe filter for both archive notes and archived page content. This disables Django’s automatic HTML escaping.

Impact  
Malicious archived pages or user notes could execute arbitrary JavaScript in a user’s browser, resulting in stored cross-site scripting.

Fix Applied  
The |safe filter was removed from all user-controlled fields in view\_archive.html.  
Django’s default auto-escaping is now enforced for archived content and notes.

Verification  
Archived content renders as text and no longer executes scripts when viewed in the application.

Additional Identified Vulnerabilities (Documented)

SQL Injection  
Location: search\_archives in vulnerable\_archive/archiver/views.py  
Issue: User input is concatenated directly into a raw SQL query string.  
Recommendation: Replace raw SQL construction with Django ORM queries to ensure proper parameterization.

Broken Access Control (IDOR)  
Location: view\_archive, edit\_archive, delete\_archive in vulnerable\_archive/archiver/views.py  
Issue: Archives are accessed by ID without enforcing ownership checks.  
Recommendation: Enforce object-level authorization by scoping queries to the authenticated user.

LLM Prompt Injection  
Location: ask\_database in vulnerable\_archive/archiver/views.py  
Issue: LLM-generated queries may be executed directly, enabling prompt injection and data exfiltration.  
Recommendation: Remove the feature or restrict behavior to allow-listed, safe operations implemented using the ORM.

Arbitrary File Write  
Location: export\_summary in vulnerable\_archive/archiver/views.py  
Issue: User-controlled filename hints allow unsafe file path generation.  
Recommendation: Generate server-side filenames and constrain file output to a fixed directory.

LLM Data Exfiltration  
Location: enrich\_archive in vulnerable\_archive/archiver/views.py and enrich\_archive.html  
Issue: Sensitive context such as private notes may be exposed to LLM processing of untrusted content.  
Recommendation: Minimize LLM prompt context, exclude sensitive fields, and avoid granting capabilities that could leak data.

Conclusion  
This capstone demonstrates practical identification, remediation, verification, and documentation of security vulnerabilities in a modern Django web application, including both traditional web vulnerabilities and risks introduced by LLM integration.

