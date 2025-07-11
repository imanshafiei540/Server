# TAK Server Development Setup Guide

This guide walks you through setting up TAK Server for development on macOS with Apple Silicon (M1/M2).

## Prerequisites

### Required Software
- **Java 17** (required, not Java 8)
- **Docker** (for PostgreSQL database)
- **Git** (for version control)

### Install Java 17
```bash
# Install Java 17 using Homebrew
brew install openjdk@17

# Add to PATH
export PATH="/opt/homebrew/opt/openjdk@17/bin:$PATH"

# Verify installation
java -version
```

## Initial Setup

### 1. Clone and Prepare Repository
```bash
git clone <repository-url>
cd Server
```

### 2. Fix Apple Silicon Compatibility
The default protobuf version doesn't work with Apple Silicon. Update `src/takserver-protobuf/build.gradle`:

```gradle
// Change these lines:
protobufVersion = '3.21.12'  // was 3.0.0
grpcVersion = '1.60.0'       // was 1.0.2
```

### 3. Create Git Tag (Required for Build)
```bash
git tag -a 4.8.0 -m "Initial tag for development"
```

### 4. Build TAK Server
```bash
cd src
./gradlew clean bootWar bootJar shadowJar
```

## Database Setup

### 1. Start PostgreSQL Container
```bash
docker run -it -d --rm --name TakserverServer0DB \
    --env POSTGRES_PASSWORD=e815f795745e \
    --env POSTGRES_HOST_AUTH_METHOD=trust \
    --env POSTGRES_USER=martiuser \
    --env POSTGRES_DB=cot \
    -p 5432:5432 postgis/postgis:15-3.3
```

### 2. Initialize Database Schema
```bash
cd src/takserver-schemamanager
../gradlew shadowJar
java -jar build/libs/schemamanager-*-uber.jar upgrade
```

## Configuration

### 1. Create CoreConfig.xml
```bash
cd src/takserver-core/example
cp CoreConfig.example.xml CoreConfig.xml
```

Update the database connection in `CoreConfig.xml`:
```xml
<connection url="jdbc:postgresql://127.0.0.1:5432/cot" username="martiuser" password="e815f795745e" />
```

### 2. Generate Certificates
```bash
cd certs
export STATE="California"
export CITY="San Francisco"
export ORGANIZATION="Development"
export ORGANIZATIONAL_UNIT="IT"

# Generate root CA
./makeRootCa.sh

# Generate server certificate
./makeCert.sh server takserver

# Generate client certificate (optional)
./makeCert.sh client admin
```

### 3. Create User Authentication File
Create `UserAuthenticationFile.xml` with proper BCrypt hashed passwords:

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<UserAuthenticationFile xmlns="http://bbn.com/marti/xml/bindings">
    <User identifier="admin" password="[BCRYPT_HASH]" passwordHashed="true" role="ROLE_ADMIN">
        <groupList>__ANON__</groupList>
        <groupList>__ADMIN__</groupList>
        <groupList>__WEBTAK__</groupList>
    </User>
</UserAuthenticationFile>
```

**Generate BCrypt Hash:**
- Use online tool: https://bcrypt-generator.com/
- Enter password: `atakatak`
- Cost factor: `10`
- Copy the generated hash

## Running TAK Server

### 1. Create Start Script
Create `start-takserver.sh`:

```bash
#!/bin/bash

export IGNITE_HOME="$PWD/ignite"
export JDK_JAVA_OPTIONS="-Dloader.path=WEB-INF/lib-provided,WEB-INF/lib,WEB-INF/classes,file:lib/ -Djava.net.preferIPv4Stack=true -Djava.security.egd=file:/dev/./urandom -DIGNITE_UPDATE_NOTIFIER=false -DIGNITE_QUIET=true -Dio.netty.tmpdir=$PWD -Djava.io.tmpdir=$PWD -Dio.netty.native.workdir=$PWD -Djdk.tls.client.protocols=TLSv1.2 --add-opens=java.base/sun.security.pkcs=ALL-UNNAMED --add-opens=java.base/sun.security.pkcs10=ALL-UNNAMED --add-opens=java.base/sun.security.util=ALL-UNNAMED --add-opens=java.base/sun.security.x509=ALL-UNNAMED --add-opens=java.base/sun.security.tools.keytool=ALL-UNNAMED --add-opens=java.base/jdk.internal.misc=ALL-UNNAMED --add-opens=java.base/sun.nio.ch=ALL-UNNAMED --add-opens=java.management/com.sun.jmx.mbeanserver=ALL-UNNAMED --add-opens=jdk.internal.jvmstat/sun.jvmstat.monitor=ALL-UNNAMED --add-opens=java.base/sun.reflect.generics.reflectiveObjects=ALL-UNNAMED --add-opens=jdk.management/com.sun.management.internal=ALL-UNNAMED --add-opens=java.base/java.io=ALL-UNNAMED --add-opens=java.base/java.nio=ALL-UNNAMED --add-opens=java.base/java.util=ALL-UNNAMED --add-opens=java.base/java.util.concurrent=ALL-UNNAMED --add-opens=java.base/java.util.concurrent.locks=ALL-UNNAMED --add-opens=java.base/java.util.concurrent.atomic=ALL-UNNAMED --add-opens=java.base/java.lang=ALL-UNNAMED --add-opens=java.base/java.lang.invoke=ALL-UNNAMED --add-opens=java.base/java.math=ALL-UNNAMED --add-opens=java.sql/java.sql=ALL-UNNAMED --add-opens=java.base/javax.net.ssl=ALL-UNNAMED --add-opens=java.base/java.net=ALL-UNNAMED --add-opens=jdk.unsupported/sun.misc=ALL-UNNAMED --add-opens=java.base/java.lang.ref=ALL-UNNAMED --add-opens=java.base/java.lang.reflect=ALL-UNNAMED --add-opens=java.base/java.security=ALL-UNNAMED --add-opens=java.base/java.security.ssl=ALL-UNNAMED --add-opens=java.base/java.security.cert=ALL-UNNAMED --add-opens=java.base/sun.security.rsa=ALL-UNNAMED --add-opens=java.base/sun.security.ssl=ALL-UNNAMED --add-opens=java.base/sun.security.x500=ALL-UNNAMED --add-opens=java.base/sun.security.pkcs12=ALL-UNNAMED --add-opens=java.base/sun.security.provider=ALL-UNNAMED --add-opens=java.base/javax.security.auth.x500=ALL-UNNAMED"

echo "Starting TAK Server..."
echo "Configuration: https://localhost:8443"
echo "API: https://localhost:8443/swagger-ui.html"
echo "No client certs: https://localhost:8446"
echo ""

echo "Starting Configuration microservice..."
java -server -XX:+AlwaysPreTouch -XX:+UseG1GC -XX:+ScavengeBeforeFullGC -XX:+DisableExplicitGC -Xmx2g \
-Dspring.profiles.active=config,duplicatelogs \
-jar ../build/libs/takserver-core-*.war &

sleep 5

echo "Starting Messaging microservice..."
java -server -XX:+AlwaysPreTouch -XX:+UseG1GC -XX:+ScavengeBeforeFullGC -XX:+DisableExplicitGC -Xmx2g \
-Dspring.profiles.active=messaging,duplicatelogs \
-jar ../build/libs/takserver-core-*.war &

sleep 5

echo "Starting API microservice..."
java -server -XX:+AlwaysPreTouch -XX:+UseG1GC -XX:+ScavengeBeforeFullGC -XX:+DisableExplicitGC -Xmx2g \
-Dspring.profiles.active=api,duplicatelogs \
-Dkeystore.pkcs12.legacy \
-jar ../build/libs/takserver-core-*.war &

echo ""
echo "TAK Server is starting up..."
echo "It may take 1-2 minutes for all services to be ready."
echo ""
echo "Default credentials: admin / atakatak"
```

### 2. Start TAK Server
```bash
chmod +x start-takserver.sh
./start-takserver.sh
```

## Accessing TAK Server

### Setup Browser Certificate Trust

1. **Add to /etc/hosts:**
   ```
   127.0.0.1 takserver
   ```

2. **Install root CA in macOS Keychain:**
   ```bash
   sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain certs/files/ca.pem
   ```

### Access URLs

- **Admin Interface (no client certs):** https://takserver:8446/
- **API with client certs:** https://takserver:8443/swagger-ui.html
- **Metrics Dashboard:** https://takserver:8446/metrics/
- **Client Dashboard:** https://takserver:8446/clients/

### Default Login
- **Username:** admin
- **Password:** atakatak

## Architecture

TAK Server consists of 3 microservices:
1. **Configuration Service** - Centralized configuration management
2. **Messaging Service** - Handles client connections and message routing
3. **API Service** - REST API and web interface

## Key Ports

- **8443** - HTTPS with client certificate authentication
- **8446** - HTTPS without client certificates (admin interface)
- **8089** - TAK client connections (TLS)
- **5432** - PostgreSQL database

## Troubleshooting

### Common Issues

1. **"No user found" errors:** Password must be BCrypt hashed with `passwordHashed="true"`
2. **Certificate errors:** Make sure hostname matches certificate (use `takserver` not `localhost`)
3. **Build failures:** Ensure Java 17 is installed and protobuf version is updated for Apple Silicon
4. **Database connection:** Verify PostgreSQL container is running and accessible

### Logs
Check logs in the `logs/` directory:
- `takserver-config.log`
- `takserver-messaging.log`
- `takserver-api.log`

## Development Notes

### Files NOT in Version Control
These files are excluded from Git for security/environment reasons:
- `UserAuthenticationFile.xml` - Contains sensitive authentication data
- `start-takserver.sh` - Environment-specific runtime script
- `certs/` - Certificate files
- `logs/` - Log files
- `build/` - Build artifacts

### Important Configuration
- Database password: `e815f795745e`
- Database user: `martiuser`
- Database name: `cot`
- Default admin password: `atakatak` (change in production!)

## Next Steps

1. **Create additional users** via the User Management interface
2. **Configure missions** and data sharing
3. **Set up client connections** using the generated certificates
4. **Explore the API** via Swagger UI
5. **Customize configuration** for your specific needs

## Resources

- [TAK Server Configuration Guide](src/docs/TAK_Server_Configuration_Guide.pdf)
- [Test Architecture and Development](src/takserver-takcl-core/docs/Development.md)
- [Publishing Guide](src/docs/publishing.md)

---

## Manual Start Commands (Alternative to Script)

If you prefer to start services manually in separate terminals (useful for debugging):

### Prerequisites
- Ensure database is running: `docker ps` should show `TakserverServer0DB`
- Navigate to: `cd src/takserver-core/example`
- Set environment variables (run in each terminal):

```bash
export IGNITE_HOME="$PWD/ignite"
export JDK_JAVA_OPTIONS="-Dloader.path=WEB-INF/lib-provided,WEB-INF/lib,WEB-INF/classes,file:lib/ -Djava.net.preferIPv4Stack=true -Djava.security.egd=file:/dev/./urandom -DIGNITE_UPDATE_NOTIFIER=false -DIGNITE_QUIET=true -Dio.netty.tmpdir=$PWD -Djava.io.tmpdir=$PWD -Dio.netty.native.workdir=$PWD -Djdk.tls.client.protocols=TLSv1.2  --add-opens=java.base/sun.security.pkcs=ALL-UNNAMED --add-opens=java.base/sun.security.pkcs10=ALL-UNNAMED --add-opens=java.base/sun.security.util=ALL-UNNAMED --add-opens=java.base/sun.security.x509=ALL-UNNAMED --add-opens=java.base/sun.security.tools.keytool=ALL-UNNAMED --add-opens=java.base/jdk.internal.misc=ALL-UNNAMED --add-opens=java.base/sun.nio.ch=ALL-UNNAMED --add-opens=java.management/com.sun.jmx.mbeanserver=ALL-UNNAMED --add-opens=jdk.internal.jvmstat/sun.jvmstat.monitor=ALL-UNNAMED --add-opens=java.base/sun.reflect.generics.reflectiveObjects=ALL-UNNAMED --add-opens=jdk.management/com.sun.management.internal=ALL-UNNAMED --add-opens=java.base/java.io=ALL-UNNAMED --add-opens=java.base/java.nio=ALL-UNNAMED --add-opens=java.base/java.util=ALL-UNNAMED --add-opens=java.base/java.util.concurrent=ALL-UNNAMED --add-opens=java.base/java.util.concurrent.locks=ALL-UNNAMED --add-opens=java.base/java.util.concurrent.atomic=ALL-UNNAMED --add-opens=java.base/java.lang=ALL-UNNAMED --add-opens=java.base/java.lang.invoke=ALL-UNNAMED --add-opens=java.base/java.math=ALL-UNNAMED --add-opens=java.sql/java.sql=ALL-UNNAMED --add-opens=java.base/javax.net.ssl=ALL-UNNAMED --add-opens=java.base/java.net=ALL-UNNAMED --add-opens=jdk.unsupported/sun.misc=ALL-UNNAMED --add-opens=java.base/java.lang.ref=ALL-UNNAMED --add-opens=java.base/java.lang.reflect=ALL-UNNAMED --add-opens=java.base/java.security=ALL-UNNAMED --add-opens=java.base/java.security.ssl=ALL-UNNAMED --add-opens=java.base/java.security.cert=ALL-UNNAMED --add-opens=java.base/sun.security.rsa=ALL-UNNAMED --add-opens=java.base/sun.security.ssl=ALL-UNNAMED --add-opens=java.base/sun.security.x500=ALL-UNNAMED --add-opens=java.base/sun.security.pkcs12=ALL-UNNAMED --add-opens=java.base/sun.security.provider=ALL-UNNAMED --add-opens=java.base/javax.security.auth.x500=ALL-UNNAMED"
```

### Terminal 1: Configuration Service
```bash
java -server -XX:+AlwaysPreTouch -XX:+UseG1GC -XX:+ScavengeBeforeFullGC -XX:+DisableExplicitGC -Xmx2g -Dspring.profiles.active=config,duplicatelogs -jar ../build/libs/takserver-core-4.8.0-null-0.war
```

### Terminal 2: Messaging Service (wait for config to start)
```bash
java -server -XX:+AlwaysPreTouch -XX:+UseG1GC -XX:+ScavengeBeforeFullGC -XX:+DisableExplicitGC -Xmx2g -Dspring.profiles.active=messaging,duplicatelogs -jar ../build/libs/takserver-core-4.8.0-null-0.war
```

### Terminal 3: API Service (wait for messaging to start)
```bash
java -server -XX:+AlwaysPreTouch -XX:+UseG1GC -XX:+ScavengeBeforeFullGC -XX:+DisableExplicitGC -Xmx2g -Dspring.profiles.active=api,duplicatelogs -Dkeystore.pkcs12.legacy -jar ../build/libs/takserver-core-4.8.0-null-0.war
```

### Quick Stop Commands
```bash
# Find and kill TAK Server processes
ps aux | grep takserver-core | grep -v grep | awk '{print $2}' | xargs kill -9

# Or kill specific processes by PID
kill -9 [PID1] [PID2] [PID3]

# Stop database container (if needed)
docker stop TakserverServer0DB
```

### Notes for Manual Start
- **Start in order**: Config → Messaging → API
- **Wait between starts**: Let each service fully initialize before starting the next
- **Check logs**: Monitor `logs/` directory for startup progress
- **Memory**: Adjust `-Xmx2g` based on your system resources
- **JAR filename**: Update `takserver-core-4.8.0-null-0.war` to match your built version
