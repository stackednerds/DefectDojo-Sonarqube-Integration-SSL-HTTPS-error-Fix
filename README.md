# DefectDojo-Sonarqube-Integration-SSL-HTTPS-error-Fix


**Created by:** Stacked Nerds  
**Date:** 2025-02-24 11:21:48 UTC

## Prerequisites
- DefectDojo installed on Server 1 (e.g., `/dd/django-DefectDojo`)

## Steps to Resolve SSL Verification Error

### 1. Create Certificate Directory on DefectDojo Server
```bash
# On DefectDojo Server
cd /dd/django-DefectDojo
mkdir -p certs/sonarqube
```

### 2. Copy SSL Certificates from SonarQube Server
Copy the SSL certificates from the SonarQube server to the DefectDojo server using a secure method (e.g., manual copy, secure file transfer).

#### Manual Certificate Export
```bash
# On SonarQube Server
cat /dd/sq/ssl/server.crt
# Copy the output

# On DefectDojo Server
vim /dd/django-DefectDojo/certs/sonarqube/server.crt
# Paste the certificate content
```

### 3. Create SSL Certificate Configuration File
Create a Python file to handle SSL verification in the `docker/extra_settings` directory:

```bash
# On DefectDojo Server
vim /dd/django-DefectDojo/docker/extra_settings/disable_ssl_verify.py
```

Add the following content to the file:
```python name=docker/extra_settings/disable_ssl_verify.py
import os
import requests
import urllib3
from urllib3.util.ssl_ import create_urllib3_context

# Point to the mounted SonarQube certificate
CERT_PATH = "/sonarqube-ssl/server.crt"

if os.path.exists(CERT_PATH):
    # Configure requests to use the certificate
    os.environ['REQUESTS_CA_BUNDLE'] = CERT_PATH
    urllib3.disable_warnings()
else:
    print(f"Certificate not found at {CERT_PATH}")
    urllib3.disable_warnings()
    requests.packages.urllib3.disable_warnings()

# Patch the SonarQube API client
try:
    from dojo.tools.sonarqube.api_client import SonarQubeAPI
    original_init = SonarQubeAPI.__init__
    
    def new_init(self, *args, **kwargs):
        original_init(self, *args, **kwargs)
        if os.path.exists(CERT_PATH):
            self.session.verify = CERT_PATH
        else:
            self.session.verify = False
    
    SonarQubeAPI.__init__ = new_init
except Exception as e:
    print(f"Failed to patch SonarQubeAPI: {str(e)}")
```

### 4. Modify docker-compose.yml
Update the docker-compose.yml in `/dd/django-DefectDojo` to use the local certificate path and include the extra settings volume:
```yaml name=docker-compose.yml
  uwsgi:
    # ... existing configurations ...
    environment:
      # ... existing environment variables ...
      REQUESTS_CA_BUNDLE: "/sonarqube-ssl/server.crt"
    volumes:
        - type: bind
          source: ./docker/extra_settings
          target: /app/docker/extra_settings
        - type: bind
          source: ./certs/sonarqube  # Local path to certificates
          target: /sonarqube-ssl
        - "defectdojo_media:${DD_MEDIA_ROOT:-/app/media}"

  celeryworker:
    # ... existing configurations ...
    environment:
      # ... existing environment variables ...
      REQUESTS_CA_BUNDLE: "/sonarqube-ssl/server.crt"
    volumes:
        - type: bind
          source: ./docker/extra_settings
          target: /app/docker/extra_settings
        - type: bind
          source: ./certs/sonarqube  # Local path to certificates
          target: /sonarqube-ssl
        - "defectdojo_media:${DD_MEDIA_ROOT:-/app/media}"
```

### 5. Set Proper Permissions
```bash
# On DefectDojo Server
chmod 644 /dd/django-DefectDojo/certs/sonarqube/server.crt
```

### 6. Restart DefectDojo Services
```bash
cd /dd/django-DefectDojo
docker compose down
docker compose up -d
```

### 7. Verify Certificate Installation
```bash
# Check if certificates are mounted correctly
docker compose exec uwsgi ls -l /sonarqube-ssl/

# Verify certificate content
docker compose exec uwsgi openssl x509 -in /sonarqube-ssl/server.crt -text -noout
```

### 8. Configure SonarQube Integration in DefectDojo
1. Log into DefectDojo web interface
2. Go to Configuration > Tool Configuration
3. Click "Add Tool Configuration"
4. Select "SonarQube" as the tool type
5. Fill in:
   - Name: Your preferred name
   - URL: Your SonarQube URL (https://...)
   - API Key: Your SonarQube token

## Verification Commands
```bash
# Verify certificate validity
openssl verify /dd/django-DefectDojo/certs/sonarqube/server.crt

# Check certificate expiration
openssl x509 -in /dd/django-DefectDojo/certs/sonarqube/server.crt -noout -dates

# Test SonarQube connectivity
curl --cacert /dd/django-DefectDojo/certs/sonarqube/server.crt https://your-sonarqube-server

# Check DefectDojo logs
docker compose logs uwsgi | grep -i "ssl"
```

## Important Notes
1. Always use secure methods to transfer certificates.
2. Ensure proper firewall rules between servers.
3. Monitor certificate expiration dates.

For further assistance, please refer to the official DefectDojo and SonarQube documentation or contact support.
