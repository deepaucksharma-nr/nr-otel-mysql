# Diagnosis Report: core-metrics

**Generated:** Sun Jul 20 17:17:07 IST 2025
**Module:** core-metrics
**Port:** 8081
**Health Port:** 13133

## Summary

- ❌ Module has health issues
- ❌ Configuration has issues
- ❌ Some expected metrics are missing

## Recommended Actions

1. **If module is not running:**
   ```bash
   cd modules/core-metrics
   docker-compose up -d
   ```

2. **If metrics are missing:**
   - Check MySQL connectivity
   - Verify configuration syntax
   - Review container logs

3. **If New Relic export is failing:**
   - Verify NEW_RELIC_LICENSE_KEY
   - Check NEW_RELIC_OTLP_ENDPOINT
   - Review network connectivity

4. **For federation issues:**
   - Ensure dependent modules are running
   - Verify service endpoint configuration

## Useful Commands

```bash
# Check container status
docker ps | grep core-metrics

# View container logs
docker logs core-metrics-otel-collector

# Test health endpoint
curl http://localhost:13133/

# Test metrics endpoint  
curl http://localhost:8081/metrics

# Restart module
cd modules/core-metrics && docker-compose restart
```

