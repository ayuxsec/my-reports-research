https://hackerone.com/reports/530974

## Normal Flow

```mermaid
flowchart LR
    A[User Browser] --> B[Creative UI]
    B --> C[Import Image]
    C --> D[API media import]
    D --> E[Validate URL]
    E --> F[Fetch Remote Image]
    F --> G[Store Media]
    G --> H[Attach To Creative]
```

* User supplies external image URL
* Backend validates destination
* Backend fetches image
* Image stored and bound to creative

## Vulnerable Flow

```mermaid
flowchart LR
    A[User Browser] --> B[Creative UI]
    B --> C[Import Image]
    C --> D[API media import]
    D --> E[Fetch Remote Image]
    E --> F[Internal Network Access]
    F --> G[Store Media]
    G --> H[Attach To Creative]
```

* URL accepted without validation
* Backend fetches arbitrary host
* Internal services reachable

## Attack Path

```mermaid
flowchart LR
    A[Attacker] --> B[Controlled Domain]
    B --> C[API media import]
    C --> D[DNS Rebind]
    D --> E[Metadata Service]
    E --> F[Credentials Response]
    F --> G[Stored As Media]
```

* Attacker controls initial DNS resolution
* DNS changed to link local metadata IP
* Backend reuses resolved IP
* Sensitive metadata returned

## Vulnerable Backend Logic

```go
func importMedia(url string) error {
    resp, err := http.Get(url)
    if err != nil {
        return err
    }
    data, _ := io.ReadAll(resp.Body)
    storeMedia(data)
    return nil
}
```

* Missing security check

  * No allowlist or denylist for IP ranges
  * No revalidation after DNS resolution
  * No block on link local or metadata IPs

* Resulting system state change

  * Backend performs authenticated requests to internal metadata service
  * SSH keys and instance metadata exposed to attacker
