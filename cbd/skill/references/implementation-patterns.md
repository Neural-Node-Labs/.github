# Implementation Patterns — Phase II Reference

Language-specific patterns for try-catch, dual-stream logging, circuit breakers, and schema
validation. Use these during Phase II implementation to ensure every component meets the
Stability & Fault-Tolerance Standard.

---

## Python

### Component Shell (Standard Pattern)

```python
import logging
import json
from datetime import datetime, timezone
from dataclasses import dataclass, asdict
from typing import Any

# ── Dual-Stream Logger Setup ─────────────────────────────────────────────────
system_log = logging.getLogger("system")
llm_trace_log = logging.getLogger("llm_trace")

def trace(phase: str, component: str, state: dict):
    """Write to llm_interaction.log — call at every significant state change."""
    llm_trace_log.info(json.dumps({
        "timestamp": datetime.now(timezone.utc).isoformat(),
        "phase": phase,
        "component": component,
        **state
    }))

# ── Error Schema ─────────────────────────────────────────────────────────────
def error_schema(code: str, message: str, component: str) -> dict:
    return {
        "code": code,
        "message": message,
        "component": component,
        "timestamp": datetime.now(timezone.utc).isoformat()
    }

# ── Component Template ────────────────────────────────────────────────────────
class InvoiceValidator:
    """
    Reason for Existence: Validate that an invoice payload meets business rules
    before downstream processing.

    IN:  {invoiceId: str, amount: float, currency: str, lineItems: list}
    OUT: {invoiceId: str, valid: bool, errors: list[str]}
    ERR: {code: "VALIDATION_ERROR", message: str, component: "InvoiceValidator", timestamp: str}
    """

    _failure_count = 0
    _CIRCUIT_THRESHOLD = 3
    _circuit_open = False

    def validate(self, payload: dict) -> dict:
        # Circuit breaker check
        if self._circuit_open:
            system_log.error("InvoiceValidator circuit is OPEN — returning safe default")
            return {"invoiceId": payload.get("invoiceId"), "valid": False, "errors": ["SERVICE_UNAVAILABLE"]}

        # IN-Schema defensive validation
        required = ["invoiceId", "amount", "currency", "lineItems"]
        missing = [f for f in required if f not in payload]
        if missing:
            return error_schema("INVALID_IN_SCHEMA", f"Missing fields: {missing}", "InvoiceValidator")

        try:
            trace("PHASE_II", "InvoiceValidator", {"event": "validation_start", "invoiceId": payload["invoiceId"]})

            errors = []
            if payload["amount"] <= 0:
                errors.append("Amount must be positive")
            if payload["currency"] not in ["USD", "EUR", "GBP"]:
                errors.append(f"Unsupported currency: {payload['currency']}")
            if not payload["lineItems"]:
                errors.append("lineItems cannot be empty")

            result = {"invoiceId": payload["invoiceId"], "valid": len(errors) == 0, "errors": errors}

            trace("PHASE_II", "InvoiceValidator", {"event": "validation_complete", "valid": result["valid"], "errorCount": len(errors)})

            self._failure_count = 0  # reset on success
            return result

        except Exception as e:
            self._failure_count += 1
            system_log.exception(f"InvoiceValidator failure #{self._failure_count}: {e}")

            if self._failure_count >= self._CIRCUIT_THRESHOLD:
                self._circuit_open = True
                system_log.critical("InvoiceValidator circuit OPENED after repeated failures")

            return error_schema("VALIDATION_EXCEPTION", str(e), "InvoiceValidator")

    def reset(self):
        """Self-healing: call to close circuit and reset failure counter."""
        self._failure_count = 0
        self._circuit_open = False
        system_log.info("InvoiceValidator circuit reset")
```

---

## TypeScript / Node.js

### Component Shell (Standard Pattern)

```typescript
import * as fs from "fs";

// ── Dual-Stream Logger ────────────────────────────────────────────────────────
const systemLog = fs.createWriteStream("system.log", { flags: "a" });
const llmTraceLog = fs.createWriteStream("llm_interaction.log", { flags: "a" });

function logSystem(level: string, component: string, message: string, meta?: object) {
  const entry = JSON.stringify({ level, component, message, timestamp: new Date().toISOString(), ...meta });
  systemLog.write(entry + "\n");
  if (level === "ERROR") console.error(`[${component}] ${message}`);
}

function trace(phase: string, component: string, state: object) {
  const entry = JSON.stringify({ timestamp: new Date().toISOString(), phase, component, ...state });
  llmTraceLog.write(entry + "\n");
}

// ── Error Schema ──────────────────────────────────────────────────────────────
function errorSchema(code: string, message: string, component: string) {
  return { code, message, component, timestamp: new Date().toISOString() };
}

// ── Component ─────────────────────────────────────────────────────────────────
class PaymentProcessor {
  /**
   * Reason for Existence: Execute a single payment transaction against the payment gateway.
   * IN:  { orderId: string, amount: number, currency: string, paymentToken: string }
   * OUT: { orderId: string, transactionId: string, status: "success" | "declined" }
   * ERR: { code: string, message: string, component: "PaymentProcessor", timestamp: string }
   */

  private failureCount = 0;
  private readonly CIRCUIT_THRESHOLD = 3;
  private circuitOpen = false;

  async process(payload: {
    orderId: string; amount: number; currency: string; paymentToken: string;
  }): Promise<object> {
    if (this.circuitOpen) {
      logSystem("ERROR", "PaymentProcessor", "Circuit OPEN — rejecting request");
      return errorSchema("CIRCUIT_OPEN", "Payment service temporarily disabled", "PaymentProcessor");
    }

    // IN-Schema validation
    const required = ["orderId", "amount", "currency", "paymentToken"];
    const missing = required.filter(k => !(k in payload));
    if (missing.length > 0) {
      return errorSchema("INVALID_IN_SCHEMA", `Missing: ${missing.join(", ")}`, "PaymentProcessor");
    }

    try {
      trace("PHASE_II", "PaymentProcessor", { event: "payment_start", orderId: payload.orderId, amount: payload.amount });

      // ← Implementation logic goes here →
      const transactionId = `TXN-${Date.now()}`;
      const status = "success"; // real gateway call replaces this

      const result = { orderId: payload.orderId, transactionId, status };

      trace("PHASE_II", "PaymentProcessor", { event: "payment_complete", orderId: payload.orderId, status });

      this.failureCount = 0;
      return result;

    } catch (err: any) {
      this.failureCount++;
      logSystem("ERROR", "PaymentProcessor", `Failure #${this.failureCount}: ${err.message}`, { stack: err.stack });

      if (this.failureCount >= this.CIRCUIT_THRESHOLD) {
        this.circuitOpen = true;
        logSystem("CRITICAL", "PaymentProcessor", "Circuit OPENED");
      }

      return errorSchema("PAYMENT_EXCEPTION", err.message, "PaymentProcessor");
    }
  }

  reset() {
    this.failureCount = 0;
    this.circuitOpen = false;
    logSystem("INFO", "PaymentProcessor", "Circuit reset");
  }
}
```

---

## Go

### Component Shell (Standard Pattern)

```go
package components

import (
    "encoding/json"
    "fmt"
    "log"
    "os"
    "sync/atomic"
    "time"
)

// ── Dual-Stream Loggers ───────────────────────────────────────────────────────
var (
    systemLog   = log.New(openLogFile("system.log"), "", 0)
    llmTraceLog = log.New(openLogFile("llm_interaction.log"), "", 0)
)

func openLogFile(name string) *os.File {
    f, err := os.OpenFile(name, os.O_APPEND|os.O_CREATE|os.O_WRONLY, 0644)
    if err != nil {
        log.Fatalf("cannot open %s: %v", name, err)
    }
    return f
}

func traceLog(phase, component string, state map[string]interface{}) {
    state["timestamp"] = time.Now().UTC().Format(time.RFC3339)
    state["phase"] = phase
    state["component"] = component
    b, _ := json.Marshal(state)
    llmTraceLog.Println(string(b))
}

// ── Error Schema ──────────────────────────────────────────────────────────────
type ErrorSchema struct {
    Code      string `json:"code"`
    Message   string `json:"message"`
    Component string `json:"component"`
    Timestamp string `json:"timestamp"`
}

func newError(code, message, component string) ErrorSchema {
    return ErrorSchema{code, message, component, time.Now().UTC().Format(time.RFC3339)}
}

// ── Component ─────────────────────────────────────────────────────────────────
// FileParser: Parse a single JSON file into a typed domain record.
// IN:  {filePath: string}
// OUT: {records: []DomainRecord, parseErrors: int}
// ERR: ErrorSchema
type FileParser struct {
    failureCount int64
    circuitOpen  int32 // atomic: 0=closed, 1=open
}

type ParseResult struct {
    Records     []map[string]interface{} `json:"records"`
    ParseErrors int                      `json:"parseErrors"`
}

func (fp *FileParser) Parse(filePath string) (interface{}, error) {
    if atomic.LoadInt32(&fp.circuitOpen) == 1 {
        systemLog.Println(`{"level":"ERROR","component":"FileParser","message":"circuit OPEN"}`)
        return newError("CIRCUIT_OPEN", "FileParser temporarily disabled", "FileParser"), nil
    }

    if filePath == "" {
        return newError("INVALID_IN_SCHEMA", "filePath is required", "FileParser"), nil
    }

    traceLog("PHASE_II", "FileParser", map[string]interface{}{"event": "parse_start", "filePath": filePath})

    data, err := os.ReadFile(filePath)
    if err != nil {
        return fp.handleError(fmt.Sprintf("read failed: %v", err))
    }

    var records []map[string]interface{}
    parseErrors := 0
    if jsonErr := json.Unmarshal(data, &records); jsonErr != nil {
        parseErrors++
    }

    result := ParseResult{Records: records, ParseErrors: parseErrors}
    traceLog("PHASE_II", "FileParser", map[string]interface{}{"event": "parse_complete", "recordCount": len(records), "parseErrors": parseErrors})

    atomic.StoreInt64(&fp.failureCount, 0)
    return result, nil
}

func (fp *FileParser) handleError(msg string) (interface{}, error) {
    count := atomic.AddInt64(&fp.failureCount, 1)
    systemLog.Printf(`{"level":"ERROR","component":"FileParser","message":"%s","failureCount":%d}`, msg, count)
    if count >= 3 {
        atomic.StoreInt32(&fp.circuitOpen, 1)
        systemLog.Println(`{"level":"CRITICAL","component":"FileParser","message":"circuit OPENED"}`)
    }
    return newError("PARSE_EXCEPTION", msg, "FileParser"), nil
}

func (fp *FileParser) Reset() {
    atomic.StoreInt64(&fp.failureCount, 0)
    atomic.StoreInt32(&fp.circuitOpen, 0)
    systemLog.Println(`{"level":"INFO","component":"FileParser","message":"circuit reset"}`)
}
```

---

## Adaptor Pattern (Language-Agnostic Rules)

An Adaptor is **pure transformation logic** — no I/O, no business rules, no API calls.

```python
# CORRECT Adaptor — maps fields only
class CRM_To_Invoice_Adaptor:
    def adapt(self, crm_payload: dict) -> dict:
        try:
            return {
                "invoiceId": crm_payload["opportunity_id"],     # field rename
                "amount": float(crm_payload["deal_value"]),     # type coercion
                "currency": crm_payload.get("currency", "USD"), # default fill
                "lineItems": crm_payload.get("products", [])    # safe default
            }
        except (KeyError, ValueError, TypeError) as e:
            return error_schema("ADAPT_FAIL", str(e), "CRM_To_Invoice_Adaptor")

# WRONG — an Adaptor must NEVER do this:
class BadAdaptor:
    def adapt(self, payload):
        result = requests.get("https://api.crm.com/validate", json=payload)  # NO — I/O forbidden
        if result["amount"] < 0: raise ValueError("negative amount")         # NO — business rule
        return result
```
