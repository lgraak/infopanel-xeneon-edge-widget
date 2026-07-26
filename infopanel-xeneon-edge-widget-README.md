# Use InfoPanel Full-Screen on the Corsair Xeneon Edge in iCUE Widget Mode

This guide documents a working method for displaying an InfoPanel dashboard full-screen on the Corsair Xeneon Edge while using iCUE widget mode.

The main benefit is that InfoPanel can use nearly the entire Xeneon Edge display without the Windows taskbar consuming space at the bottom.

This is an unofficial workaround and is not affiliated with Corsair or InfoPanel.

## Overview

The working path is:

```text
InfoPanel web server
    ↓
HTTPS reverse proxy
    ↓
iCUE iFrame widget
    ↓
Corsair Xeneon Edge
```

InfoPanel renders the selected panel through its built-in web server. iCUE then loads that page inside a full-screen iFrame widget.

## Requirements

- Corsair Xeneon Edge
- Corsair iCUE with widget mode enabled
- InfoPanel
- InfoPanel web server enabled
- An HTTPS reverse proxy
- A hostname that resolves to the reverse proxy
- Windows Firewall access from the reverse proxy to the InfoPanel PC

This guide uses Nginx Proxy Manager, but another HTTPS-capable reverse proxy should also work.

## 1. Create the InfoPanel profile

Create the dashboard in InfoPanel at:

```text
2560 × 720
```

Using the native panel dimensions avoids the blur caused by CSS transform scaling.

Editing is easiest while the Xeneon Edge is in normal monitor mode. After saving the profile, switch the display back to iCUE widget mode.

## 2. Enable the InfoPanel web server

In InfoPanel:

1. Open **Settings**
2. Open the web server settings
3. Enable the web server
4. Bind it to the IP address of the Windows PC
5. Select a listening port
6. Set the refresh interval

Example:

```text
Listen address: 192.0.2.10
Port: 80
Refresh interval: 500 ms
```

A 500 ms interval produces two updates per second.

Lower values provide faster updates, but may increase CPU usage. Around 200 to 500 ms is a reasonable starting range.

## 3. Identify the profile URL

InfoPanel exposes profiles using zero-based numeric paths.

```text
Profile 1 = /0
Profile 2 = /1
Profile 3 = /2
Profile 4 = /3
Profile 5 = /4
Profile 6 = /5
Profile 7 = /6
Profile 8 = /7
```

For example, profile 8 would be available at:

```text
http://192.0.2.10/7
```

Replace the IP address and profile number with your own values.

## 4. Allow access through Windows Firewall

The reverse proxy must be able to reach the InfoPanel web server.

Example PowerShell rule:

```powershell
New-NetFirewallRule `
    -DisplayName "InfoPanel Web Server TCP 80" `
    -Direction Inbound `
    -Protocol TCP `
    -LocalPort 80 `
    -LocalAddress 192.0.2.10 `
    -RemoteAddress 192.0.2.20 `
    -Profile Private `
    -Action Allow
```

In this example:

```text
192.0.2.10 = InfoPanel PC
192.0.2.20 = reverse proxy
```

Verify that InfoPanel is listening:

```powershell
Get-NetTCPConnection -LocalPort 80 -State Listen
```

Test from the reverse proxy:

```bash
curl -sS -o /dev/null -w '%{http_code}\n' http://192.0.2.10/7
```

Expected result:

```text
200
```

A `405 Method Not Allowed` response from `curl -I` is not necessarily a failure. `curl -I` sends a HEAD request, while InfoPanel may allow only GET.

## 5. Configure the HTTPS reverse proxy

The iCUE embedded browser did not load the direct HTTP URL reliably. Putting InfoPanel behind HTTPS resolved the issue.

Example Nginx Proxy Manager configuration:

```text
Domain Names: infopanel.example.com
Scheme: http
Forward Hostname / IP: 192.0.2.10
Forward Port: 80
```

Recommended options:

```text
Block Common Exploits: Enabled
Websockets Support: Enabled
Force SSL: Enabled
HTTP/2 Support: Enabled
```

Add a DNS record:

```text
infopanel.example.com → reverse proxy IP
```

Verify the proxied page in a normal browser:

```text
https://infopanel.example.com/7
```

The page should load without a certificate warning.

## 6. Create the iCUE iFrame widget

In iCUE widget mode, create an iFrame widget and use:

```html
<div style="position:absolute;inset:0;overflow:hidden;background:#050708;">
  <iframe
    src="https://infopanel.example.com/7"
    style="position:absolute;inset:0;width:100%;height:100%;border:0;margin:0;padding:0;overflow:hidden;background:#050708;"
    scrolling="no">
  </iframe>
</div>
```

Change the final profile number as needed:

```text
/0 through /7
```

The same template can be reused for every full-screen InfoPanel profile.

## 7. Avoid CSS transform scaling

This approach caused visible blur:

```css
transform: scale(0.832);
```

The blur occurred because the rendered InfoPanel output was being resampled.

For the sharpest result:

- Use a 2560 × 720 InfoPanel canvas
- Use `width:100%`
- Use `height:100%`
- Do not use `transform: scale()`

## iCUE viewport behavior

The iCUE widget browser reported values similar to:

```text
innerWidth: 2114
innerHeight: 581
devicePixelRatio: 1.2
```

The effective physical size is approximately:

```text
2114 × 1.2 ≈ 2537
581 × 1.2 ≈ 697
```

This closely matches the usable widget region on the Xeneon Edge.

The important part is to let the iCUE browser handle its own device-pixel mapping instead of applying manual CSS transform scaling.

## Editing workflow

The main limitation is that editing the InfoPanel layout is inconvenient while the Xeneon Edge is in iCUE widget mode.

A practical workflow is:

1. Switch the Xeneon Edge to normal monitor mode
2. Edit the InfoPanel profile
3. Save the profile
4. Switch back to widget mode
5. Reload the iFrame widget if necessary

The iFrame code does not need to be recreated when the panel content changes.

## Troubleshooting

### `ERR_CONNECTION_REFUSED`

Confirm that:

- The full URL includes `http://` or `https://`
- InfoPanel is listening on the expected address and port
- Windows Firewall permits the connection
- The reverse proxy can reach the InfoPanel host

### `ERR_SSL_PROTOCOL_ERROR`

This can occur when iCUE attempts to load plain HTTP content as HTTPS.

Use a proper HTTPS reverse proxy and load the proxied URL instead.

### Blank iFrame

Check the InfoPanel response headers:

```powershell
$response = Invoke-WebRequest http://192.0.2.10/7
$response.Headers | Format-List
```

Look for:

```text
X-Frame-Options
Content-Security-Policy
```

If neither is present, embedding is probably not being blocked by InfoPanel.

Also verify that the HTTPS URL loads in a normal browser without certificate errors.

### Content is clipped

Confirm that:

- The InfoPanel canvas is 2560 × 720
- The iFrame uses `width:100%`
- The iFrame uses `height:100%`
- No CSS transform scaling is applied
- The widget is using the largest available iCUE layout

### Content is blurry

Remove any CSS such as:

```css
transform: scale(...);
zoom: ...;
```

Use the native InfoPanel canvas dimensions and allow iCUE to perform the final device-pixel mapping.

## Reusable template

```html
<div style="position:absolute;inset:0;overflow:hidden;background:#050708;">
  <iframe
    src="https://infopanel.example.com/PROFILE_NUMBER"
    style="position:absolute;inset:0;width:100%;height:100%;border:0;margin:0;padding:0;overflow:hidden;background:#050708;"
    scrolling="no">
  </iframe>
</div>
```

Replace:

```text
PROFILE_NUMBER
```

with:

```text
0 through 7
```

## Result

This setup provides:

- Full-screen InfoPanel rendering in iCUE widget mode
- No Windows taskbar on the Xeneon Edge
- Sharp native-looking output
- Reusable profile URLs
- No need for native Xeneon Edge support inside InfoPanel

Native integration could still simplify setup, but the built-in InfoPanel web server, HTTPS reverse proxy, and iCUE iFrame widget are enough to make the system work.
