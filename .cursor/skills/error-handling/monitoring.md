# Monitoring & Alerting Setup

This document covers monitoring error patterns and setting up alerts for production issues.

## Error Rate Monitoring

### Tracking Error Frequencies

```typescript
// src/common/metrics/error-tracker.ts
import { Injectable } from '@nestjs/common';

interface ErrorMetrics {
  total: number;
  by4xx: number;
  by5xx: number;
  byType: Record<string, number>;
  lastHour: number[];
}

@Injectable()
export class ErrorTracker {
  private metrics: ErrorMetrics = {
    total: 0,
    by4xx: 0,
    by5xx: 0,
    byType: {},
    lastHour: [],
  };

  recordError(statusCode: number, errorType: string): void {
    this.metrics.total++;

    if (statusCode < 500) {
      this.metrics.by4xx++;
    } else {
      this.metrics.by5xx++;
    }

    this.metrics.byType[errorType] = (this.metrics.byType[errorType] || 0) + 1;

    // Track last hour for alerting
    const now = Math.floor(Date.now() / 60000); // Minute buckets
    this.metrics.lastHour.push(now);

    // Keep only last 60 minutes
    this.metrics.lastHour = this.metrics.lastHour.filter(
      t => now - t < 60
    );
  }

  getMetrics() {
    return {
      ...this.metrics,
      errorRatePerMinute: this.metrics.lastHour.length,
      error4xxPercent: Math.round((this.metrics.by4xx / this.metrics.total) * 100),
      error5xxPercent: Math.round((this.metrics.by5xx / this.metrics.total) * 100),
    };
  }

  reset(): void {
    this.metrics = {
      total: 0,
      by4xx: 0,
      by5xx: 0,
      byType: {},
      lastHour: [],
    };
  }
}
```

### Metrics Endpoint

```typescript
@Controller('health')
export class HealthController {
  constructor(private readonly errorTracker: ErrorTracker) {}

  @Get('metrics')
  getMetrics() {
    return this.errorTracker.getMetrics();
  }

  @Get('errors/summary')
  getErrorSummary() {
    const metrics = this.errorTracker.getMetrics();

    return {
      totalErrors: metrics.total,
      errors4xx: metrics.by4xx,
      errors5xx: metrics.by5xx,
      topErrors: Object.entries(metrics.byType)
        .sort(([, a], [, b]) => b - a)
        .slice(0, 10)
        .map(([type, count]) => ({ type, count })),
      errorRatePerMinute: metrics.errorRatePerMinute,
    };
  }
}
```

## Alert Thresholds

### Configuration

```typescript
// src/config/alert.config.ts
export const alertThresholds = {
  // Error rates (errors per minute)
  critical5xxRate: 10,      // 10+ 5xx errors per minute
  warning5xxRate: 5,        // 5-9 5xx errors per minute
  critical4xxRate: 50,      // 50+ 4xx errors per minute
  warning4xxRate: 20,       // 20-49 4xx errors per minute

  // Error types
  criticalErrors: [
    'DatabaseError',
    'ExternalServiceError',
    'PaymentFailedError',
    'InternalServerError',
  ],

  // Response times
  slowRequestThreshold: 5000, // 5 seconds
  criticalSlowRequestRate: 10, // 10% of requests
};
```

### Alert Service

```typescript
// src/common/services/alert.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { alertThresholds } from '../config/alert.config';

@Injectable()
export class AlertService {
  private readonly logger = new Logger(AlertService.name);

  checkErrorMetrics(metrics: any): void {
    // Check 5xx error rate
    if (metrics.error5xxPercent > 50) {
      this.sendCriticalAlert(
        'HIGH_5XX_ERROR_RATE',
        `5xx error rate: ${metrics.error5xxPercent}%`,
        metrics
      );
    } else if (metrics.by5xx > alertThresholds.warning5xxRate) {
      this.sendWarningAlert(
        'ELEVATED_5XX_RATE',
        `5xx errors: ${metrics.by5xx} in last minute`,
        metrics
      );
    }

    // Check 4xx error rate
    if (metrics.by4xx > alertThresholds.critical4xxRate) {
      this.sendWarningAlert(
        'HIGH_4XX_RATE',
        `4xx errors: ${metrics.by4xx} in last minute`,
        metrics
      );
    }

    // Check for critical error types
    for (const errorType of alertThresholds.criticalErrors) {
      const count = metrics.byType[errorType] || 0;
      if (count > 5) {
        this.sendWarningAlert(
          `CRITICAL_ERROR_DETECTED`,
          `${errorType}: ${count} occurrences`,
          { errorType, count }
        );
      }
    }
  }

  private sendCriticalAlert(
    code: string,
    message: string,
    context: any
  ): void {
    this.logger.error(`🚨 CRITICAL ALERT: ${message}`, {
      alertCode: code,
      context,
      timestamp: new Date().toISOString(),
    });

    // Send to monitoring service (PagerDuty, etc)
    this.notifyOncall(code, message, 'critical', context);
  }

  private sendWarningAlert(
    code: string,
    message: string,
    context: any
  ): void {
    this.logger.warn(`⚠️ WARNING: ${message}`, {
      alertCode: code,
      context,
      timestamp: new Date().toISOString(),
    });

    // Send to monitoring service
    this.notifyOncall(code, message, 'warning', context);
  }

  private notifyOncall(
    code: string,
    message: string,
    severity: string,
    context: any
  ): void {
    // Implement based on your monitoring platform
    // Examples: PagerDuty, OpsGenie, Slack, etc.
    if (process.env.ALERT_WEBHOOK_URL) {
      fetch(process.env.ALERT_WEBHOOK_URL, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          code,
          message,
          severity,
          context,
          timestamp: new Date().toISOString(),
        }),
      }).catch(err => {
        this.logger.error('Failed to send alert', err);
      });
    }
  }
}
```

## Error Patterns & Anomalies

### Pattern Detection

```typescript
// src/common/services/error-anomaly-detector.ts
@Injectable()
export class ErrorAnomalyDetector {
  private readonly logger = new Logger(ErrorAnomalyDetector.name);
  private errorHistory: Record<string, number[]> = {};

  recordError(errorType: string, timestamp: number): void {
    if (!this.errorHistory[errorType]) {
      this.errorHistory[errorType] = [];
    }

    this.errorHistory[errorType].push(timestamp);

    // Keep only last hour
    const oneHourAgo = timestamp - 60 * 60 * 1000;
    this.errorHistory[errorType] = this.errorHistory[errorType].filter(
      t => t > oneHourAgo
    );

    this.detectAnomalies(errorType);
  }

  private detectAnomalies(errorType: string): void {
    const timestamps = this.errorHistory[errorType];

    if (timestamps.length < 5) return;

    // Calculate average time between errors
    const intervals = [];
    for (let i = 1; i < timestamps.length; i++) {
      intervals.push(timestamps[i] - timestamps[i - 1]);
    }

    const avgInterval = intervals.reduce((a, b) => a + b) / intervals.length;
    const lastInterval = timestamps[timestamps.length - 1] - timestamps[timestamps.length - 2];

    // Spike detection: if last interval is significantly smaller than average
    if (lastInterval < avgInterval * 0.5) {
      this.logger.warn(`Anomaly detected: ${errorType} spike`, {
        errorType,
        frequency: timestamps.length,
        avgInterval: Math.round(avgInterval),
        lastInterval: Math.round(lastInterval),
      });

      // Alert on spike
      if (timestamps.length > 10) {
        this.logger.error(`Critical spike: ${errorType}`, {
          errorType,
          occurrences: timestamps.length,
        });
      }
    }
  }
}
```

## Health Check Implementation

### Liveness Probe

```typescript
// src/health/liveness.controller.ts
import { Controller, Get } from '@nestjs/common';

@Controller('health')
export class HealthController {
  @Get('live')
  liveness() {
    return {
      status: 'ok',
      timestamp: new Date().toISOString(),
      uptime: process.uptime(),
    };
  }
}
```

### Readiness Probe

```typescript
@Controller('health')
export class ReadinessController {
  constructor(
    private readonly db: Database,
    private readonly cache: Cache,
  ) {}

  @Get('ready')
  async readiness() {
    try {
      // Check database
      await this.db.query('SELECT 1');

      // Check cache
      await this.cache.ping();

      return {
        status: 'ready',
        database: 'ok',
        cache: 'ok',
        timestamp: new Date().toISOString(),
      };
    } catch (error) {
      return {
        status: 'not_ready',
        error: error.message,
        timestamp: new Date().toISOString(),
      };
    }
  }
}
```

## Distributed Tracing

### Request ID Propagation

```typescript
// src/common/middleware/trace-id.middleware.ts
import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';
import { v4 as uuidv4 } from 'uuid';

@Injectable()
export class TraceIdMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    const traceId =
      req.headers['x-trace-id'] ||
      req.headers['x-request-id'] ||
      uuidv4();

    req.id = traceId;

    // Propagate in response
    res.setHeader('X-Trace-ID', traceId);

    next();
  }
}
```

### Trace Context in All Logs

```typescript
@Injectable()
export class TraceIdService {
  private readonly asyncLocalStorage = new AsyncLocalStorage<string>();

  run<T>(traceId: string, callback: () => T): T {
    return this.asyncLocalStorage.run(traceId, callback);
  }

  getTraceId(): string {
    return this.asyncLocalStorage.getStore() || 'unknown';
  }
}
```

## Performance Baseline Monitoring

### Endpoint Performance Tracking

```typescript
// src/common/interceptors/performance.interceptor.ts
@Injectable()
export class PerformanceInterceptor implements NestInterceptor {
  private readonly logger = new Logger(PerformanceInterceptor.name);

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest<Request>();
    const startTime = Date.now();

    return next.handle().pipe(
      tap(() => {
        const duration = Date.now() - startTime;
        const method = request.method;
        const path = request.path;

        // Log performance
        this.logger.debug(`${method} ${path}`, {
          duration,
          slow: duration > 1000,
        });

        // Alert on slow endpoints
        if (duration > 5000) {
          this.logger.warn(`SLOW ENDPOINT: ${method} ${path}`, { duration });
        }
      }),
      catchError(error => {
        const duration = Date.now() - startTime;

        this.logger.error(
          `${request.method} ${request.path} failed`,
          {
            duration,
            error: error.message,
          }
        );

        throw error;
      })
    );
  }
}
```

## Dead Letter Queue for Failed Events

```typescript
// src/common/services/dead-letter-queue.ts
@Injectable()
export class DeadLetterQueue {
  private readonly logger = new Logger(DeadLetterQueue.name);

  async enqueue(message: any, error: Error, retryCount: number = 0): Promise<void> {
    const maxRetries = 3;

    if (retryCount < maxRetries) {
      // Retry with exponential backoff
      const delayMs = Math.pow(2, retryCount) * 1000;

      this.logger.warn(
        `Requeuing message (attempt ${retryCount + 1}/${maxRetries})`,
        {
          delayMs,
          messageType: message.type,
          error: error.message,
        }
      );

      setTimeout(() => {
        this.processMessage(message, retryCount + 1);
      }, delayMs);
    } else {
      // Persist to DLQ
      this.logger.error(
        `Message moved to DLQ after ${maxRetries} retries`,
        {
          messageType: message.type,
          error: error.message,
          message: JSON.stringify(message),
        }
      );

      // Save to database for manual inspection
      await this.saveToDLQ(message, error, retryCount);
    }
  }

  private async saveToDLQ(message: any, error: Error, retryCount: number): Promise<void> {
    // Implement based on your database/storage
  }

  private async processMessage(message: any, retryCount: number): Promise<void> {
    // Implement message processing
  }
}
```

## SLA Monitoring

```typescript
// src/common/services/sla-monitor.ts
export class SLAMonitor {
  private readonly slas = {
    'GET /users': { p99: 200, p95: 100 }, // milliseconds
    'POST /users': { p99: 500, p95: 300 },
    'POST /orders': { p99: 1000, p95: 500 },
  };

  recordLatency(method: string, path: string, latency: number): void {
    const key = `${method} ${path}`;
    const sla = this.slas[key];

    if (!sla) return;

    if (latency > sla.p99) {
      console.warn(`SLA VIOLATION (p99): ${key} - ${latency}ms > ${sla.p99}ms`);
    } else if (latency > sla.p95) {
      console.warn(`SLA WARNING (p95): ${key} - ${latency}ms > ${sla.p95}ms`);
    }
  }
}
```
