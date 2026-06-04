# Logging Strategies

## Overview

Logging is the practice of recording application events, errors, and state changes to understand system behavior, debug issues, and monitor performance. Effective logging strategies combine structured logging, appropriate log levels, centralized aggregation, and efficient search capabilities to provide comprehensive observability into application health and behavior.

## Table of Contents
- [Log Levels](#log-levels)
- [Structured Logging](#structured-logging)
- [Logging Libraries](#logging-libraries)
- [Log Formatting](#log-formatting)
- [Centralized Logging](#centralized-logging)
- [Log Aggregation](#log-aggregation)
- [ELK Stack](#elk-stack)
- [Cloud Logging Solutions](#cloud-logging-solutions)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [When to Use/Not to Use](#when-to-usenot-to-use)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Log Levels

### Standard Log Levels

```javascript
// Log level hierarchy (lowest to highest)
const LogLevels = {
  TRACE: 10,   // Finest-grained diagnostic information
  DEBUG: 20,   // Detailed information for debugging
  INFO: 30,    // General informational messages
  WARN: 40,    // Warning messages (potential issues)
  ERROR: 50,   // Error messages (application errors)
  FATAL: 60,   // Critical errors (application crashes)
};

// Example usage
class Logger {
  constructor(level = 'INFO') {
    this.level = LogLevels[level];
  }
  
  trace(message, meta = {}) {
    if (this.level <= LogLevels.TRACE) {
      console.log(`[TRACE] ${message}`, meta);
    }
  }
  
  debug(message, meta = {}) {
    if (this.level <= LogLevels.DEBUG) {
      console.log(`[DEBUG] ${message}`, meta);
    }
  }
  
  info(message, meta = {}) {
    if (this.level <= LogLevels.INFO) {
      console.info(`[INFO] ${message}`, meta);
    }
  }
  
  warn(message, meta = {}) {
    if (this.level <= LogLevels.WARN) {
      console.warn(`[WARN] ${message}`, meta);
    }
  }
  
  error(message, meta = {}) {
    if (this.level <= LogLevels.ERROR) {
      console.error(`[ERROR] ${message}`, meta);
    }
  }
  
  fatal(message, meta = {}) {
    if (this.level <= LogLevels.FATAL) {
      console.error(`[FATAL] ${message}`, meta);
    }
  }
}

const logger = new Logger(process.env.LOG_LEVEL || 'INFO');

// Usage examples
logger.trace('Entering function calculateTotal', { items: 5 });
logger.debug('Database query executed', { query: 'SELECT * FROM users', duration: '45ms' });
logger.info('User logged in successfully', { userId: '12345', ip: '192.168.1.1' });
logger.warn('API rate limit approaching', { current: 950, limit: 1000 });
logger.error('Failed to save user data', { userId: '12345', error: err.message });
logger.fatal('Database connection lost', { host: 'db.example.com' });
```

### When to Use Each Level

```javascript
// TRACE - Very detailed diagnostic information
logger.trace('Function entry', { 
  function: 'calculateDiscount',
  arguments: { subtotal: 100, discountCode: 'SAVE20' }
});

// DEBUG - Detailed information for debugging
logger.debug('Cache hit', { 
  key: 'user:12345',
  ttl: 3600,
  size: '2.3KB'
});

// INFO - General informational messages
logger.info('Order placed successfully', {
  orderId: 'ORD-123456',
  userId: '12345',
  total: 99.99
});

// WARN - Warning messages
logger.warn('High memory usage detected', {
  current: '85%',
  threshold: '80%',
  action: 'garbage collection triggered'
});

// ERROR - Error messages
logger.error('Payment processing failed', {
  orderId: 'ORD-123456',
  paymentMethod: 'credit_card',
  error: 'Card declined',
  errorCode: 'CARD_DECLINED'
});

// FATAL - Critical errors
logger.fatal('Application crash imminent', {
  reason: 'Out of memory',
  memoryUsage: '98%',
  action: 'Restarting application'
});
```

## Structured Logging

### JSON Structured Logs

```javascript
// Basic structured log entry
const logEntry = {
  timestamp: new Date().toISOString(),
  level: 'INFO',
  message: 'User authentication successful',
  context: {
    userId: '12345',
    username: 'john.doe@example.com',
    ip: '192.168.1.100',
    userAgent: 'Mozilla/5.0...',
    sessionId: 'sess_abc123',
  },
  metadata: {
    service: 'auth-service',
    version: '1.2.3',
    environment: 'production',
    hostname: 'api-server-01',
    pid: 12345,
  },
};

console.log(JSON.stringify(logEntry));
```

### Advanced Structured Logger

```typescript
// structured-logger.ts
interface LogContext {
  [key: string]: any;
}

interface LogEntry {
  timestamp: string;
  level: string;
  message: string;
  context?: LogContext;
  error?: {
    name: string;
    message: string;
    stack?: string;
  };
  trace?: {
    traceId?: string;
    spanId?: string;
    parentSpanId?: string;
  };
  service: {
    name: string;
    version: string;
    environment: string;
  };
  host: {
    hostname: string;
    pid: number;
  };
}

class StructuredLogger {
  private serviceName: string;
  private version: string;
  private environment: string;
  private defaultContext: LogContext;
  
  constructor(config: {
    serviceName: string;
    version: string;
    environment: string;
    defaultContext?: LogContext;
  }) {
    this.serviceName = config.serviceName;
    this.version = config.version;
    this.environment = config.environment;
    this.defaultContext = config.defaultContext || {};
  }
  
  private createLogEntry(
    level: string,
    message: string,
    context?: LogContext,
    error?: Error
  ): LogEntry {
    const entry: LogEntry = {
      timestamp: new Date().toISOString(),
      level,
      message,
      context: { ...this.defaultContext, ...context },
      service: {
        name: this.serviceName,
        version: this.version,
        environment: this.environment,
      },
      host: {
        hostname: require('os').hostname(),
        pid: process.pid,
      },
    };
    
    if (error) {
      entry.error = {
        name: error.name,
        message: error.message,
        stack: error.stack,
      };
    }
    
    // Add trace context if available (OpenTelemetry)
    if ((global as any).activeSpan) {
      const span = (global as any).activeSpan;
      entry.trace = {
        traceId: span.traceId,
        spanId: span.spanId,
        parentSpanId: span.parentSpanId,
      };
    }
    
    return entry;
  }
  
  private write(entry: LogEntry): void {
    console.log(JSON.stringify(entry));
  }
  
  info(message: string, context?: LogContext): void {
    this.write(this.createLogEntry('INFO', message, context));
  }
  
  warn(message: string, context?: LogContext): void {
    this.write(this.createLogEntry('WARN', message, context));
  }
  
  error(message: string, error?: Error, context?: LogContext): void {
    this.write(this.createLogEntry('ERROR', message, context, error));
  }
  
  debug(message: string, context?: LogContext): void {
    this.write(this.createLogEntry('DEBUG', message, context));
  }
  
  child(additionalContext: LogContext): StructuredLogger {
    return new StructuredLogger({
      serviceName: this.serviceName,
      version: this.version,
      environment: this.environment,
      defaultContext: { ...this.defaultContext, ...additionalContext },
    });
  }
}

// Usage
const logger = new StructuredLogger({
  serviceName: 'user-service',
  version: '1.2.3',
  environment: process.env.NODE_ENV || 'development',
});

// Log with context
logger.info('User created', {
  userId: '12345',
  email: 'user@example.com',
  role: 'admin',
});

// Log error with context
try {
  throw new Error('Database connection failed');
} catch (error) {
  logger.error('Failed to save user', error as Error, {
    userId: '12345',
    operation: 'create',
  });
}

// Create child logger with additional context
const requestLogger = logger.child({
  requestId: 'req_abc123',
  userId: '12345',
  path: '/api/users',
});

requestLogger.info('Request received');
requestLogger.info('Request completed', { duration: '45ms' });
```

## Logging Libraries

### Winston (Node.js)

```javascript
// winston-setup.js
const winston = require('winston');
const { format, transports } = winston;

// Custom format for development
const devFormat = format.combine(
  format.colorize(),
  format.timestamp({ format: 'YYYY-MM-DD HH:mm:ss' }),
  format.printf(({ timestamp, level, message, ...meta }) => {
    let msg = `${timestamp} [${level}]: ${message}`;
    if (Object.keys(meta).length > 0) {
      msg += ` ${JSON.stringify(meta)}`;
    }
    return msg;
  })
);

// JSON format for production
const prodFormat = format.combine(
  format.timestamp(),
  format.errors({ stack: true }),
  format.json()
);

const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: process.env.NODE_ENV === 'production' ? prodFormat : devFormat,
  defaultMeta: {
    service: 'user-service',
    version: process.env.APP_VERSION || '1.0.0',
    environment: process.env.NODE_ENV || 'development',
  },
  transports: [
    // Console output
    new transports.Console(),
    
    // File transports
    new transports.File({ 
      filename: 'logs/error.log', 
      level: 'error',
      maxsize: 10485760, // 10MB
      maxFiles: 5,
    }),
    new transports.File({ 
      filename: 'logs/combined.log',
      maxsize: 10485760,
      maxFiles: 10,
    }),
  ],
  exceptionHandlers: [
    new transports.File({ filename: 'logs/exceptions.log' }),
  ],
  rejectionHandlers: [
    new transports.File({ filename: 'logs/rejections.log' }),
  ],
});

// Add HTTP transport for centralized logging
if (process.env.LOG_ENDPOINT) {
  logger.add(new transports.Http({
    host: process.env.LOG_ENDPOINT,
    path: '/logs',
    ssl: true,
  }));
}

module.exports = logger;

// Usage
const logger = require('./winston-setup');

logger.info('User logged in', {
  userId: '12345',
  email: 'user@example.com',
});

logger.error('Payment failed', {
  orderId: 'ORD-123',
  amount: 99.99,
  error: error.message,
});
```

### Pino (High-Performance Logger)

```javascript
// pino-setup.js
const pino = require('pino');

const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  
  // Pretty print in development
  transport: process.env.NODE_ENV === 'development' ? {
    target: 'pino-pretty',
    options: {
      colorize: true,
      translateTime: 'SYS:standard',
      ignore: 'pid,hostname',
    },
  } : undefined,
  
  // Base fields
  base: {
    service: 'user-service',
    version: process.env.APP_VERSION,
    environment: process.env.NODE_ENV,
  },
  
  // Serializers for common objects
  serializers: {
    req: pino.stdSerializers.req,
    res: pino.stdSerializers.res,
    err: pino.stdSerializers.err,
  },
  
  // Redact sensitive fields
  redact: {
    paths: ['password', 'authorization', 'cookie', '*.password', '*.token'],
    remove: true,
  },
});

// Usage
logger.info({ userId: '12345' }, 'User logged in');
logger.error({ err: error, orderId: 'ORD-123' }, 'Payment failed');

// Express middleware
const expressPino = require('express-pino-logger');
app.use(expressPino({ logger }));
```

### React Logging

```typescript
// logger.ts
class BrowserLogger {
  private isDev = process.env.NODE_ENV === 'development';
  
  private send(level: string, message: string, context?: any): void {
    const logEntry = {
      timestamp: new Date().toISOString(),
      level,
      message,
      context,
      url: window.location.href,
      userAgent: navigator.userAgent,
      sessionId: this.getSessionId(),
    };
    
    // Console in development
    if (this.isDev) {
      console[level.toLowerCase()](message, context);
    }
    
    // Send to backend in production
    if (!this.isDev && level !== 'DEBUG') {
      this.sendToBackend(logEntry);
    }
  }
  
  private getSessionId(): string {
    let sessionId = sessionStorage.getItem('sessionId');
    if (!sessionId) {
      sessionId = `sess_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
      sessionStorage.setItem('sessionId', sessionId);
    }
    return sessionId;
  }
  
  private async sendToBackend(logEntry: any): Promise<void> {
    try {
      await fetch('/api/logs', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(logEntry),
        keepalive: true, // Important for logs sent during page unload
      });
    } catch (error) {
      // Fail silently to avoid logging loops
    }
  }
  
  info(message: string, context?: any): void {
    this.send('INFO', message, context);
  }
  
  warn(message: string, context?: any): void {
    this.send('WARN', message, context);
  }
  
  error(message: string, error?: Error, context?: any): void {
    this.send('ERROR', message, {
      ...context,
      error: error ? {
        name: error.name,
        message: error.message,
        stack: error.stack,
      } : undefined,
    });
  }
  
  debug(message: string, context?: any): void {
    this.send('DEBUG', message, context);
  }
}

export const logger = new BrowserLogger();

// Usage in React component
import React, { useEffect } from 'react';
import { logger } from './logger';

export const UserProfile: React.FC = () => {
  useEffect(() => {
    logger.info('UserProfile component mounted');
    
    return () => {
      logger.info('UserProfile component unmounted');
    };
  }, []);
  
  const handleSave = async () => {
    try {
      logger.info('Saving user profile', { userId: '12345' });
      await saveProfile();
      logger.info('User profile saved successfully');
    } catch (error) {
      logger.error('Failed to save profile', error as Error, {
        userId: '12345',
      });
    }
  };
  
  return <div>User Profile</div>;
};
```

### Angular Logging Service

```typescript
// logging.service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { environment } from '../environments/environment';

export enum LogLevel {
  Debug = 0,
  Info = 1,
  Warn = 2,
  Error = 3,
}

interface LogEntry {
  timestamp: string;
  level: string;
  message: string;
  context?: any;
  url?: string;
  userAgent?: string;
}

@Injectable({
  providedIn: 'root'
})
export class LoggingService {
  private logLevel: LogLevel = environment.production ? LogLevel.Info : LogLevel.Debug;
  private logQueue: LogEntry[] = [];
  private flushInterval = 5000; // 5 seconds
  
  constructor(private http: HttpClient) {
    this.startFlushTimer();
  }
  
  private createLogEntry(level: string, message: string, context?: any): LogEntry {
    return {
      timestamp: new Date().toISOString(),
      level,
      message,
      context,
      url: window.location.href,
      userAgent: navigator.userAgent,
    };
  }
  
  private shouldLog(level: LogLevel): boolean {
    return level >= this.logLevel;
  }
  
  private log(level: LogLevel, levelName: string, message: string, context?: any): void {
    if (!this.shouldLog(level)) return;
    
    const entry = this.createLogEntry(levelName, message, context);
    
    // Console output
    if (!environment.production) {
      const consoleMethod = levelName.toLowerCase() as 'debug' | 'info' | 'warn' | 'error';
      console[consoleMethod](message, context);
    }
    
    // Queue for backend
    if (level >= LogLevel.Info) {
      this.logQueue.push(entry);
      
      // Immediate flush for errors
      if (level === LogLevel.Error) {
        this.flush();
      }
    }
  }
  
  private startFlushTimer(): void {
    setInterval(() => this.flush(), this.flushInterval);
  }
  
  private flush(): void {
    if (this.logQueue.length === 0) return;
    
    const logs = [...this.logQueue];
    this.logQueue = [];
    
    this.http.post('/api/logs/batch', { logs }).subscribe({
      error: (err) => {
        // Restore logs on error
        this.logQueue.unshift(...logs);
      }
    });
  }
  
  debug(message: string, context?: any): void {
    this.log(LogLevel.Debug, 'DEBUG', message, context);
  }
  
  info(message: string, context?: any): void {
    this.log(LogLevel.Info, 'INFO', message, context);
  }
  
  warn(message: string, context?: any): void {
    this.log(LogLevel.Warn, 'WARN', message, context);
  }
  
  error(message: string, error?: Error, context?: any): void {
    this.log(LogLevel.Error, 'ERROR', message, {
      ...context,
      error: error ? {
        name: error.name,
        message: error.message,
        stack: error.stack,
      } : undefined,
    });
  }
}

// Usage in component
import { Component, OnInit } from '@angular/core';
import { LoggingService } from './logging.service';

@Component({
  selector: 'app-user-profile',
  templateUrl: './user-profile.component.html'
})
export class UserProfileComponent implements OnInit {
  constructor(private logger: LoggingService) {}
  
  ngOnInit(): void {
    this.logger.info('UserProfile component initialized');
  }
  
  saveProfile(): void {
    try {
      this.logger.debug('Saving user profile', { userId: '12345' });
      // Save logic
      this.logger.info('User profile saved successfully');
    } catch (error) {
      this.logger.error('Failed to save profile', error as Error, {
        userId: '12345'
      });
    }
  }
}
```

## Log Formatting

### Custom Log Formats

```javascript
// Custom formatter for Winston
const customFormat = winston.format.printf(({ timestamp, level, message, ...meta }) => {
  // Format: [2024-06-15 10:30:45] INFO [user-service] User logged in userId=12345 ip=192.168.1.1
  let log = `[${timestamp}] ${level.toUpperCase()} [${meta.service || 'app'}] ${message}`;
  
  // Append key-value pairs
  Object.keys(meta).forEach(key => {
    if (key !== 'service' && key !== 'timestamp' && key !== 'level') {
      log += ` ${key}=${JSON.stringify(meta[key])}`;
    }
  });
  
  return log;
});

// Logfmt format (key=value pairs)
const logfmtFormat = winston.format.printf(({ timestamp, level, message, ...meta }) => {
  const parts = [
    `timestamp="${timestamp}"`,
    `level="${level}"`,
    `message="${message}"`,
  ];
  
  Object.keys(meta).forEach(key => {
    const value = typeof meta[key] === 'object' 
      ? JSON.stringify(meta[key]) 
      : meta[key];
    parts.push(`${key}="${value}"`);
  });
  
  return parts.join(' ');
});

// Apache Common Log Format
const apacheFormat = winston.format.printf(({ timestamp, level, message, meta }) => {
  // 192.168.1.1 - user [15/Jun/2024:10:30:45 +0000] "GET /api/users HTTP/1.1" 200 1234
  if (meta && meta.req && meta.res) {
    const req = meta.req;
    const res = meta.res;
    return `${req.ip} - ${req.user || '-'} [${timestamp}] "${req.method} ${req.url} HTTP/${req.httpVersion}" ${res.statusCode} ${res.contentLength || '-'}`;
  }
  return `[${timestamp}] ${level}: ${message}`;
});
```

## Centralized Logging

### Log Collection Architecture

```
Application Servers              Log Aggregators           Storage & Analysis
──────────────────────           ───────────────           ──────────────────

┌─────────────────┐
│   App Server 1  │──┐
│   (Winston)     │  │
└─────────────────┘  │
                     │         ┌──────────────┐
┌─────────────────┐  ├────────▶│  Filebeat/   │         ┌──────────────┐
│   App Server 2  │──┤         │  Fluentd     │────────▶│ Elasticsearch│
│   (Pino)        │  │         │  (Shipper)   │         └──────────────┘
└─────────────────┘  │         └──────────────┘                │
                     │                                          ▼
┌─────────────────┐  │         ┌──────────────┐         ┌──────────────┐
│   App Server 3  │──┘         │   Logstash   │         │    Kibana    │
│   (Custom)      │            │   (Parser)   │         │  (Visualize) │
└─────────────────┘            └──────────────┘         └──────────────┘
```

### Fluentd Configuration

```yaml
# fluentd.conf
<source>
  @type tail
  path /var/log/app/*.log
  pos_file /var/log/td-agent/app.log.pos
  tag app.logs
  <parse>
    @type json
    time_key timestamp
    time_format %Y-%m-%dT%H:%M:%S.%L%z
  </parse>
</source>

<filter app.logs>
  @type record_transformer
  <record>
    hostname ${hostname}
    environment ${ENV['ENVIRONMENT']}
  </record>
</filter>

<match app.logs>
  @type elasticsearch
  host elasticsearch.example.com
  port 9200
  index_name app-logs-%Y.%m.%d
  type_name _doc
  include_tag_key true
  tag_key @log_name
  
  <buffer>
    @type file
    path /var/log/td-agent/buffer/app
    flush_interval 10s
    flush_at_shutdown true
  </buffer>
</match>
```

### Filebeat Configuration

```yaml
# filebeat.yml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/app/*.log
    json.keys_under_root: true
    json.add_error_key: true
    fields:
      environment: production
      service: user-service
    fields_under_root: true

processors:
  - add_host_metadata:
      when.not.contains.tags: forwarded
  - add_cloud_metadata: ~
  - add_docker_metadata: ~

output.elasticsearch:
  hosts: ["elasticsearch.example.com:9200"]
  index: "app-logs-%{+yyyy.MM.dd}"
  username: "elastic"
  password: "${ELASTICSEARCH_PASSWORD}"

setup.kibana:
  host: "kibana.example.com:5601"

logging.level: info
logging.to_files: true
logging.files:
  path: /var/log/filebeat
  name: filebeat
  keepfiles: 7
  permissions: 0644
```

## Log Aggregation

### ELK Stack Setup

```yaml
# docker-compose.yml for ELK stack
version: '3.8'

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.8.0
    environment:
      - discovery.type=single-node
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - xpack.security.enabled=false
    ports:
      - "9200:9200"
    volumes:
      - elasticsearch-data:/usr/share/elasticsearch/data
  
  logstash:
    image: docker.elastic.co/logstash/logstash:8.8.0
    volumes:
      - ./logstash.conf:/usr/share/logstash/pipeline/logstash.conf
    ports:
      - "5044:5044"
      - "9600:9600"
    depends_on:
      - elasticsearch
  
  kibana:
    image: docker.elastic.co/kibana/kibana:8.8.0
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch

volumes:
  elasticsearch-data:
```

```conf
# logstash.conf
input {
  beats {
    port => 5044
  }
  
  tcp {
    port => 5000
    codec => json
  }
}

filter {
  # Parse JSON logs
  if [message] =~ /^{.*}$/ {
    json {
      source => "message"
    }
  }
  
  # Add GeoIP data for IP addresses
  if [ip] {
    geoip {
      source => "ip"
      target => "geoip"
    }
  }
  
  # Parse user agent
  if [userAgent] {
    useragent {
      source => "userAgent"
      target => "user_agent"
    }
  }
  
  # Categorize log levels
  if [level] {
    mutate {
      uppercase => [ "level" ]
    }
  }
  
  # Add timestamp
  date {
    match => [ "timestamp", "ISO8601" ]
    target => "@timestamp"
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "app-logs-%{+YYYY.MM.dd}"
  }
  
  # Debug output (remove in production)
  stdout {
    codec => rubydebug
  }
}
```

### Querying Logs with Elasticsearch

```javascript
// elasticsearch-client.js
const { Client } = require('@elastic/elasticsearch');

const client = new Client({
  node: 'http://localhost:9200',
});

// Search logs by level
async function searchByLevel(level, from = 0, size = 100) {
  const result = await client.search({
    index: 'app-logs-*',
    body: {
      query: {
        match: { level }
      },
      from,
      size,
      sort: [{ '@timestamp': 'desc' }]
    }
  });
  
  return result.hits.hits.map(hit => hit._source);
}

// Search logs by user
async function searchByUser(userId) {
  const result = await client.search({
    index: 'app-logs-*',
    body: {
      query: {
        term: { 'context.userId': userId }
      },
      sort: [{ '@timestamp': 'desc' }]
    }
  });
  
  return result.hits.hits.map(hit => hit._source);
}

// Search logs by time range
async function searchByTimeRange(startTime, endTime) {
  const result = await client.search({
    index: 'app-logs-*',
    body: {
      query: {
        range: {
          '@timestamp': {
            gte: startTime,
            lte: endTime
          }
        }
      }
    }
  });
  
  return result.hits.hits.map(hit => hit._source);
}

// Aggregate error counts by service
async function aggregateErrorsByService() {
  const result = await client.search({
    index: 'app-logs-*',
    body: {
      query: {
        match: { level: 'ERROR' }
      },
      aggs: {
        by_service: {
          terms: {
            field: 'service.name.keyword',
            size: 10
          }
        }
      },
      size: 0
    }
  });
  
  return result.aggregations.by_service.buckets;
}

// Usage
(async () => {
  const errors = await searchByLevel('ERROR');
  console.log('Recent errors:', errors);
  
  const userLogs = await searchByUser('12345');
  console.log('User activity:', userLogs);
  
  const errorStats = await aggregateErrorsByService();
  console.log('Error distribution:', errorStats);
})();
```

## Cloud Logging Solutions

### AWS CloudWatch Logs

```javascript
// cloudwatch-logger.js
const AWS = require('aws-sdk');
const cloudwatchlogs = new AWS.CloudWatchLogs({ region: 'us-east-1' });

class CloudWatchLogger {
  constructor(logGroupName, logStreamName) {
    this.logGroupName = logGroupName;
    this.logStreamName = logStreamName;
    this.sequenceToken = null;
    this.init();
  }
  
  async init() {
    try {
      await cloudwatchlogs.createLogGroup({
        logGroupName: this.logGroupName
      }).promise();
    } catch (err) {
      if (err.code !== 'ResourceAlreadyExistsException') throw err;
    }
    
    try {
      await cloudwatchlogs.createLogStream({
        logGroupName: this.logGroupName,
        logStreamName: this.logStreamName
      }).promise();
    } catch (err) {
      if (err.code !== 'ResourceAlreadyExistsException') throw err;
    }
  }
  
  async log(message, level = 'INFO', metadata = {}) {
    const logEvent = {
      message: JSON.stringify({
        timestamp: new Date().toISOString(),
        level,
        message,
        ...metadata
      }),
      timestamp: Date.now()
    };
    
    const params = {
      logEvents: [logEvent],
      logGroupName: this.logGroupName,
      logStreamName: this.logStreamName
    };
    
    if (this.sequenceToken) {
      params.sequenceToken = this.sequenceToken;
    }
    
    try {
      const response = await cloudwatchlogs.putLogEvents(params).promise();
      this.sequenceToken = response.nextSequenceToken;
    } catch (err) {
      console.error('Failed to send log to CloudWatch:', err);
    }
  }
}

// Usage
const logger = new CloudWatchLogger('my-app', 'production');
logger.log('User logged in', 'INFO', { userId: '12345' });
```

### Google Cloud Logging

```javascript
// gcp-logging.js
const { Logging } = require('@google-cloud/logging');
const logging = new Logging();

const log = logging.log('my-app-log');

function writeLog(severity, message, metadata = {}) {
  const entry = log.entry(
    {
      resource: {
        type: 'global'
      },
      severity
    },
    {
      message,
      ...metadata,
      timestamp: new Date().toISOString()
    }
  );
  
  log.write(entry);
}

// Usage
writeLog('INFO', 'User logged in', { userId: '12345' });
writeLog('ERROR', 'Payment failed', { orderId: 'ORD-123', error: 'Card declined' });
```

### Azure Application Insights

```javascript
// azure-insights.js
const appInsights = require('applicationinsights');

appInsights.setup(process.env.APPINSIGHTS_INSTRUMENTATIONKEY)
  .setAutoDependencyCorrelation(true)
  .setAutoCollectRequests(true)
  .setAutoCollectPerformance(true)
  .setAutoCollectExceptions(true)
  .setAutoCollectDependencies(true)
  .setAutoCollectConsole(true)
  .setUseDiskRetryCaching(true)
  .start();

const client = appInsights.defaultClient;

// Track custom events
client.trackEvent({ name: 'UserLoggedIn', properties: { userId: '12345' } });

// Track metrics
client.trackMetric({ name: 'DatabaseQueryTime', value: 45.2 });

// Track exceptions
try {
  throw new Error('Payment failed');
} catch (error) {
  client.trackException({ exception: error, properties: { orderId: 'ORD-123' } });
}

// Track dependencies
client.trackDependency({
  target: 'https://api.example.com',
  name: 'GET /users',
  data: '/users',
  duration: 120,
  resultCode: 200,
  success: true,
  dependencyTypeName: 'HTTP'
});
```

## Common Mistakes

### 1. Logging Sensitive Information

```javascript
// Bad: Logging sensitive data
logger.info('User login attempt', {
  email: 'user@example.com',
  password: 'secretPassword123', // NEVER log passwords
  creditCard: '4111-1111-1111-1111' // NEVER log credit cards
});

// Good: Redact sensitive information
logger.info('User login attempt', {
  email: 'user@example.com',
  userId: '12345' // Use IDs instead
});
```

### 2. Excessive Logging

```javascript
// Bad: Logging in tight loops
for (let i = 0; i < 1000000; i++) {
  logger.debug('Processing item', { index: i }); // Millions of logs!
}

// Good: Log aggregated information
logger.info('Processing started', { totalItems: 1000000 });
// Process items...
logger.info('Processing completed', { 
  totalItems: 1000000, 
  duration: '45s',
  successCount: 999500,
  errorCount: 500
});
```

### 3. Not Using Structured Logging

```javascript
// Bad: String concatenation
logger.info(`User ${userId} logged in from ${ip} at ${timestamp}`);

// Good: Structured logging
logger.info('User logged in', {
  userId,
  ip,
  timestamp
});
```

### 4. Missing Context

```javascript
// Bad: Vague log messages
logger.error('Failed');
logger.error('Something went wrong');

// Good: Detailed context
logger.error('Failed to process payment', {
  orderId: 'ORD-123',
  userId: '12345',
  paymentMethod: 'credit_card',
  amount: 99.99,
  error: error.message,
  errorCode: error.code
});
```

### 5. Synchronous Logging in Production

```javascript
// Bad: Blocking I/O for logs
function processRequest(req) {
  fs.appendFileSync('app.log', JSON.stringify(logEntry) + '\n'); // Blocks!
  // Process request
}

// Good: Asynchronous logging
function processRequest(req) {
  logger.info('Request received', { requestId: req.id }); // Non-blocking
  // Process request
}
```

## Best Practices

### 1. Use Correlation IDs

```javascript
// Express middleware for request correlation
const { v4: uuidv4 } = require('uuid');

app.use((req, res, next) => {
  req.correlationId = req.headers['x-correlation-id'] || uuidv4();
  res.setHeader('x-correlation-id', req.correlationId);
  
  req.logger = logger.child({ correlationId: req.correlationId });
  next();
});

// Usage in route
app.get('/api/users', (req, res) => {
  req.logger.info('Fetching users');
  // ...
  req.logger.info('Users fetched successfully', { count: users.length });
});
```

### 2. Implement Log Sampling

```javascript
// Sample high-volume logs
class SamplingLogger {
  constructor(logger, sampleRate = 0.1) {
    this.logger = logger;
    this.sampleRate = sampleRate;
  }
  
  shouldSample() {
    return Math.random() < this.sampleRate;
  }
  
  debug(message, context) {
    if (this.shouldSample()) {
      this.logger.debug(message, context);
    }
  }
  
  info(message, context) {
    this.logger.info(message, context);
  }
  
  error(message, error, context) {
    this.logger.error(message, error, context);
  }
}

// Sample 10% of debug logs
const samplingLogger = new SamplingLogger(logger, 0.1);
```

### 3. Set Log Retention Policies

```javascript
// Elasticsearch index lifecycle management
const ilmPolicy = {
  policy: {
    phases: {
      hot: {
        actions: {
          rollover: {
            max_age: '7d',
            max_size: '50gb'
          }
        }
      },
      warm: {
        min_age: '7d',
        actions: {
          shrink: {
            number_of_shards: 1
          },
          forcemerge: {
            max_num_segments: 1
          }
        }
      },
      cold: {
        min_age: '30d',
        actions: {
          freeze: {}
        }
      },
      delete: {
        min_age: '90d',
        actions: {
          delete: {}
        }
      }
    }
  }
};
```

### 4. Create Log Dashboards

```javascript
// Kibana dashboard configuration (simplified)
const logDashboard = {
  title: 'Application Logs Dashboard',
  panels: [
    {
      title: 'Log Level Distribution',
      type: 'pie',
      query: 'level:*',
      aggregation: 'terms',
      field: 'level.keyword'
    },
    {
      title: 'Errors Over Time',
      type: 'line',
      query: 'level:ERROR',
      aggregation: 'date_histogram',
      field: '@timestamp',
      interval: '1h'
    },
    {
      title: 'Top Error Messages',
      type: 'table',
      query: 'level:ERROR',
      aggregation: 'terms',
      field: 'message.keyword',
      size: 10
    },
    {
      title: 'Request Duration (p95)',
      type: 'metric',
      query: 'context.duration:*',
      aggregation: 'percentiles',
      field: 'context.duration',
      percentiles: [50, 95, 99]
    }
  ]
};
```

### 5. Implement Alerting

```javascript
// Elasticsearch watcher for alerts
const errorAlert = {
  trigger: {
    schedule: {
      interval: '5m'
    }
  },
  input: {
    search: {
      request: {
        indices: ['app-logs-*'],
        body: {
          query: {
            bool: {
              must: [
                { match: { level: 'ERROR' } },
                { range: { '@timestamp': { gte: 'now-5m' } } }
              ]
            }
          }
        }
      }
    }
  },
  condition: {
    compare: {
      'ctx.payload.hits.total.value': {
        gt: 10 // Alert if more than 10 errors in 5 minutes
      }
    }
  },
  actions: {
    send_email: {
      email: {
        to: 'oncall@example.com',
        subject: 'High Error Rate Detected',
        body: 'More than 10 errors detected in the last 5 minutes'
      }
    },
    slack: {
      webhook: {
        url: 'https://hooks.slack.com/services/...',
        body: JSON.stringify({
          text: 'High error rate detected in production'
        })
      }
    }
  }
};
```

## When to Use/Not to Use

### When to Use Centralized Logging

1. **Microservices Architecture**: Multiple services generating logs
2. **Distributed Systems**: Logs across multiple servers/containers
3. **Production Environments**: Need for comprehensive monitoring
4. **Compliance Requirements**: Audit trails and log retention
5. **Large Teams**: Multiple developers need log access
6. **Complex Debugging**: Correlating events across services
7. **Performance Analysis**: Identifying bottlenecks and patterns
8. **Security Monitoring**: Detecting suspicious activities

### When Local Logging is Sufficient

1. **Development Environment**: Local debugging
2. **Small Applications**: Single server, low traffic
3. **Prototypes**: Quick iteration, temporary projects
4. **Batch Jobs**: Scheduled tasks with file output
5. **Simple Scripts**: One-off utilities
6. **Budget Constraints**: No funds for logging infrastructure
7. **No Compliance Needs**: Basic logging requirements
8. **Single Developer**: No team collaboration needed

## Interview Questions

### 1. What are the main log levels and when should each be used?

**Answer**: TRACE (finest diagnostic info, rarely used), DEBUG (detailed debugging during development), INFO (general informational messages about normal operations), WARN (potentially harmful situations that don't prevent operation), ERROR (error events that allow application to continue), FATAL (severe errors causing termination). Use lower levels in development, higher levels in production to reduce noise.

### 2. What is structured logging and why is it important?

**Answer**: Structured logging formats logs as structured data (typically JSON) with consistent key-value pairs rather than plain text strings. Benefits include: easier parsing and searching, better integration with log aggregation tools, consistent format across services, ability to query specific fields, better machine readability, and support for complex filtering and analysis in tools like Elasticsearch.

### 3. How would you implement distributed tracing across microservices?

**Answer**: Use correlation IDs (request IDs) passed through all service calls via headers. Implement trace context propagation (W3C Trace Context standard or OpenTelemetry). Include trace ID, span ID, and parent span ID in all logs. Use distributed tracing tools like Jaeger, Zipkin, or cloud provider solutions. Log service boundaries, external calls, and key operations. Visualize traces to identify bottlenecks and failures.

### 4. What sensitive information should never be logged?

**Answer**: Passwords, API keys, access tokens, credit card numbers, social security numbers, personal health information (PHI), session tokens, encryption keys, private keys, OAuth secrets, and PII (personally identifiable information). Implement log redaction/masking for fields that might contain sensitive data. Use hashed or tokenized identifiers instead of raw values.

### 5. How do you handle log retention and storage costs?

**Answer**: Implement index lifecycle management with hot/warm/cold/delete phases. Use log levels to filter (ERROR/WARN in production, DEBUG in development). Implement log sampling for high-volume debug logs. Compress older logs. Archive to cheaper storage (S3 Glacier). Set retention policies based on compliance requirements. Use log aggregation to reduce duplicate entries. Monitor storage costs and adjust policies accordingly.

### 6. Explain the ELK stack and its components.

**Answer**: ELK stands for Elasticsearch, Logstash, Kibana. Elasticsearch is a search and analytics engine for storing and querying logs. Logstash is a data processing pipeline that ingests, transforms, and sends logs to Elasticsearch. Kibana is a visualization and exploration tool for creating dashboards and searching logs. Often includes Beats (lightweight data shippers like Filebeat, Metricbeat) for log collection.

### 7. How would you implement logging in a React application?

**Answer**: Create a logging service that sends logs to a backend endpoint (not directly to external services for security). Implement different log levels. Batch logs to reduce network requests. Send logs asynchronously with `fetch` and `keepalive: true`. Log user actions, errors, and performance metrics. Avoid excessive logging (don't log every render). Include session ID, user ID, and page context. Use error boundaries to catch React errors.

### 8. What's the difference between logging and monitoring?

**Answer**: Logging records discrete events (user actions, errors, state changes) as detailed records for debugging and audit trails. Monitoring tracks metrics over time (CPU, memory, request rate, error rate) for real-time health checks and alerting. Logs answer "what happened and why?", monitoring answers "is the system healthy?" Both are complementary: logs provide detail for investigation, monitoring provides high-level health status and triggers for when to investigate logs.

## Key Takeaways

1. **Use structured logging**: JSON format enables better parsing and searching
2. **Implement appropriate log levels**: DEBUG for development, INFO/WARN/ERROR for production
3. **Never log sensitive information**: Passwords, tokens, PII must be redacted
4. **Use centralized logging**: Essential for distributed systems and microservices
5. **Include context in logs**: Correlation IDs, user IDs, request IDs for tracing
6. **Implement log retention policies**: Balance compliance needs with storage costs
7. **Create meaningful log messages**: Clear, actionable information with relevant metadata
8. **Avoid excessive logging**: Performance impact and storage costs
9. **Use asynchronous logging**: Don't block application execution
10. **Monitor your logs**: Set up alerts for error spikes and anomalies

## Resources

### Logging Libraries
- [Winston](https://github.com/winstonjs/winston) - Node.js logging library
- [Pino](https://github.com/pinojs/pino) - High-performance Node.js logger
- [Bunyan](https://github.com/trentm/node-bunyan) - JSON logging for Node.js
- [Log4js](https://github.com/log4js-node/log4js-node) - Logging framework

### Log Aggregation
- [Elasticsearch](https://www.elastic.co/elasticsearch/)
- [Logstash](https://www.elastic.co/logstash/)
- [Kibana](https://www.elastic.co/kibana/)
- [Fluentd](https://www.fluentd.org/)
- [Filebeat](https://www.elastic.co/beats/filebeat)

### Cloud Solutions
- [AWS CloudWatch Logs](https://aws.amazon.com/cloudwatch/)
- [Google Cloud Logging](https://cloud.google.com/logging)
- [Azure Monitor Logs](https://azure.microsoft.com/en-us/services/monitor/)
- [Datadog Logs](https://www.datadoghq.com/product/log-management/)

### Standards
- [OpenTelemetry](https://opentelemetry.io/)
- [W3C Trace Context](https://www.w3.org/TR/trace-context/)
- [Syslog Protocol](https://tools.ietf.org/html/rfc5424)

### Articles
- [The Twelve-Factor App: Logs](https://12factor.net/logs)
- [Structured Logging Best Practices](https://www.loggly.com/blog/structured-logging-guide/)
- [Logging at Scale](https://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying)
