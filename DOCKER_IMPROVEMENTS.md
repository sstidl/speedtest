# Docker Build Process Improvements

## Analysis of Current Dockerfiles

### Issues Identified

#### Dockerfile.alpine
1. **Duplicate PHP extension installation**: Extensions are installed twice
   - First via `apk add` (php-gd, php-pdo, etc.)
   - Then via `install-php-extensions` (iconv, gd, pdo, etc.)
2. **Unnecessary tool**: The docker-php-extension-installer is downloaded but redundant since Alpine's package manager already provides everything needed

#### Dockerfile (Debian)
1. **Not using multi-stage builds**: Could reduce final image size
2. **Cleanup could be improved**: Some temporary files remain
3. **No explicit version pinning**: Could lead to reproducibility issues

### Recommended Improvements

#### 1. Optimize Alpine Dockerfile
- Remove docker-php-extension-installer (redundant with apk)
- Consolidate all apk packages in one layer
- Use multi-stage build if needed for building dependencies

#### 2. Optimize Debian Dockerfile  
- Keep docker-php-extension-installer (it's useful for Debian)
- Add multi-stage build for better layer caching
- Pin base image versions more explicitly

#### 3. General Improvements for Both
- Use BuildKit features for better caching
- Add health checks
- Optimize layer ordering (put frequently changing files last)
- Add labels for better metadata

## Implementation

### Changes Made

#### Dockerfile.alpine
1. **Fixed Alpine 3.23 compatibility**: Removed redundant Alpine PHP packages (php-*, which don't exist in Alpine 3.23) since php:8-alpine base image already includes PHP
2. **Simplified dependencies**: Only install Apache, bash, and curl from Alpine repos
3. **Use docker-php-extension-installer**: Properly install PHP extensions that work with the official PHP Docker image
4. **Added curl**: Required for health checks
5. **Added health check**: Monitor container health automatically  
6. **Added OCI labels**: Better metadata for container registries

**Before**: 3 separate RUN commands, attempting to install non-existent php-* packages
**After**: 2 RUN commands with correct package names, no redundant installations

**Note**: The original Dockerfile.alpine had a bug where it tried to install Alpine system PHP packages (like `php-gd`, `php-pdo`, etc.) which conflict with and are unnecessary for the `php:8-alpine` base image that already includes PHP.

#### Dockerfile (Debian)
1. **Consolidated cleanup**: Combined extension installation and cleanup into single layer
2. **Improved cleanup**: Added more thorough cleanup (tmp dirs, apt clean)
3. **Added health check**: Monitor container health automatically
4. **Added OCI labels**: Better metadata for container registries

**Before**: 2 separate RUN commands for install and cleanup
**After**: 1 RUN command, cleaner final image

### Benefits

1. **Smaller images**: Fewer layers, better cleanup
2. **No redundancy**: Alpine no longer installs extensions twice
3. **Better monitoring**: Health checks for container orchestration
4. **Better metadata**: OCI labels for registries and documentation
5. **Faster builds**: Fewer layers to cache and transfer

### Testing

Test the builds locally:

```bash
# Build Debian version
docker build -t librespeed:debian -f Dockerfile .

# Build Alpine version  
docker build -t librespeed:alpine -f Dockerfile.alpine .

# Test health check
docker run -d -p 8080:8080 --name speedtest librespeed:debian
docker ps  # Should show "healthy" status after 30s
```
