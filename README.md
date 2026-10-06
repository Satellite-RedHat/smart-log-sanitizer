# Smart Log Sanitizer

`parser.html` cleans a log in the browser before you share it. Paste or drop a file, choose what to remove, then copy or download the result. The log is not uploaded and is not saved. Settings stay in this browser’s local storage.

Open the file in a browser. From this directory:

```bash
xdg-open parser.html
```

Click **Load sample** to see a realistic log cleaned in place.

## Usage

1. Put a log in **Input**. Paste it, drop a text file on the card, or use **Browse**. Text logs are accepted. Images, video, audio, and PDFs are refused. Files over 32 MB are refused.
2. Leave the detectors checked, or turn off any you do not want. Add names to keep or extra strings to remove.
3. Click **Clean log**, or press Ctrl+Enter (Command+Enter on macOS).
4. Read the result under **Output**:
   - **Cleaned** is the text to share.
   - **Matches** highlights what was removed in the original text.
   - **Alias map** lists each original value when Stable aliases is selected. That map can restore the secrets. It is not part of the download.
5. **Copy** the cleaned log, or **Download** it. A file named `app.log` downloads as `app.cleaned.log`. **Copy alias map** copies the map on its own. **Clear logs** empties the input and output and leaves your settings.

**Update as you type** rewrites the output as you edit. It stays off for logs over 300,000 characters. Logs over about 80,000 characters are cleaned on a background thread. Match highlighting is hidden above 200,000 characters. The cleaned output is still complete. The size limit is 32 million characters.

**Reset settings** restores the detectors and lists. The theme and the current log stay. The theme button switches light and dark, and that choice is remembered.

## Replacement styles

| Style | Result | Use it when |
| --- | --- | --- |
| Typed labels | `[IP]`, `[HOST]`, `[EMAIL]`, and one label per kind | You want the log to stay readable |
| Stable aliases | `ip-1`, `host-2`. The same value always gets the same alias | You still need to tell two hosts or two addresses apart |
| One label | Whatever you type, default `[REDACTED]` | You want one token for everything |

With One label, an empty replacement deletes the match. Characters such as `$&` are written as themselves.

Typed labels and alias prefixes:

| Kind | Label | Alias |
| --- | --- | --- |
| IPv4 | `[IP]` | `ip-1` |
| IPv6 | `[IPV6]` | `ip6-1` |
| Hostname | `[HOST]` | `host-1` |
| Email | `[EMAIL]` | `email-1` |
| MAC | `[MAC]` | `mac-1` |
| JWT | `[JWT]` | `jwt-1` |
| Bearer token | `[TOKEN]` | `token-1` |
| Password or API key | `[SECRET]` | `secret-1` |
| AWS access key id | `[KEY]` | `aws-1` |
| Private key block | `[PRIVATE-KEY]` | `pkey-1` |
| Home directory name | `[USER]` | `user-1` |
| URL username or password | `[CRED]` | `cred-1` |
| Custom string | `[CUSTOM]` | `custom-1` |

Hostnames, emails, MAC addresses, and IPv6 addresses are matched without regard to case, so `Host.Example.COM` and `host.example.com` share one alias.

## What is removed

Each detector can be turned off.

- **IPv4.** Octets must be 0–255. A trailing CIDR prefix such as `/24` is included. `10.0.0.1:443` becomes `[IP]:443`.
- **IPv6.** Full, compressed (`2001:db8::1`, `::1`), zone ids (`fe80::1%eth0`), and IPv4-mapped (`::ffff:192.0.2.1`) forms.
- **Hostnames and domains.** Public suffixes and internal ones such as `.internal`, `.local`, `.corp`, and `.lan`. Add more under **Extra domain suffixes**, for example `corp, acme`. `example.com` on the allowlist also keeps `api.example.com`.
- **Email.** The whole address is removed, so `ada@corp.example.com` does not leave `ada@` behind.
- **MAC.** Colon, dash, and Cisco dotted forms (`00:1A:2B:3C:4D:5E`, `0011.2233.4455`).
- **JWT.** Three-part tokens that start with `eyJ`.
- **AWS access key ids.** `AKIA`, `ASIA`, `ABIA`, and `ACCA` followed by 16 characters.
- **Bearer tokens.** The token after `Bearer`. A JWT in that position keeps the `[JWT]` label.
- **Passwords and API keys.** The value after `password`, `passwd`, `pwd`, `token`, `secret`, `auth`, `api_key`, `api-key`, `access_key`, `client_secret`, or `aws_secret_access_key`, whether written with `=` or `:`. Quotes around the value stay.
- **Private keys.** PEM blocks, including OpenSSH keys.
- **Home directory names.** The username in `/home/sadas`, `/Users/Ada`, and `C:\Users\Ada`. The rest of the path stays.
- **URL usernames and passwords.** `https://deploy:s3cret@api.example.net/v1` becomes `https://[CRED]@[HOST]/v1`.
- **ANSI color codes.** Off by default. Terminal colors are stripped before the other rules run.
- **Also remove.** One string per line. Longer entries win, so `prod-server` is removed as a whole when `prod` is also listed. **Whole word** is on by default, so `log` does not remove `login`. **Case sensitive** is off by default.

## What stays

- Dates and hyphenated words, such as `2024-10-02` and `out-of-memory`.
- Filenames such as `error.log`, `Node.js`, `config.yaml`, and `foreman.service`.
- Version numbers written as `version 1.2.3.4` or `v1.2.3.4`.
- Anything on the allowlist. One entry per line. A domain keeps its subdomains. An IP is kept only as an exact match.
- A two-label name whose last label is a real suffix, such as `script.sh`, can still look like a domain. Put that filename on the allowlist if you need to keep it.
- A bare four-part number such as `9.9.9.9` is still treated as an IPv4 address.

## Enhancements

The first version removed IPv4, a narrow form of IPv6, anything that looked like a domain, and every hyphenated word, then replaced them all with `[REDACTED]`. These are the changes.

### Matching

- Hyphenated words and dates are no longer treated as hostnames. The old rule removed `2024-10-02` and `out-of-memory`.
- Filenames are no longer treated as domains. `error.log`, `Node.js`, and `foreman.service` stay.
- IPv4 octets must be 0–255. Impossible addresses such as `999.1.1.1` stay. CIDR prefixes are included. `version 1.2.3.4` and `v1.2.3.4` stay.
- IPv6 now includes compressed addresses, loopback, link-local addresses with a zone id, and IPv4-mapped addresses. A time such as `12:34:56` stays.
- Emails are removed as a whole, before the hostname rule can leave the local part behind.
- URL userinfo (`user:password@`) is removed before the host, so the password does not survive as `https://admin:s3cret@[HOST]/...`.
- Custom strings are matched literally. `$&` and `$1` in the replacement are not interpreted as regular-expression replacements.
- Custom strings are applied longest first. Whole-word and case-sensitive switches control how they match.
- Allowlisted text is protected first, including subdomains of an allowlisted domain.

### Detectors

MAC addresses, JWTs, AWS access key ids, Bearer tokens, assignment-style secrets, PEM private keys, home-directory usernames, and optional ANSI stripping were added. Each one has its own checkbox.

### Output

- Typed labels keep the kind of value visible.
- Stable aliases give each distinct value one token and show the session map on the page. The map is copied separately and is not written into the download.
- The Matches tab highlights removals and counts them.
- The download name is based on the source file.
- Copy falls back to a text selection when the clipboard API is unavailable, which includes opening the file from `file://`.

### Page

- Light and dark themes follow the toggle. The old toggle changed the icon and left the colors dark.
- Settings are stored in local storage. Log text is not.
- Status messages replace blocking alerts.
- Large logs run on a background thread.
- The page no longer loads Tailwind from a CDN, so cleaning works offline.

## Contributing

The sanitizer is one file, `parser.html`. The matching engine is the script with `id="engine"`. The page around it is the script with `id="ui"`.

To try a change, edit `parser.html`, reload it in a browser, and click **Load sample**. Check that dates, `error.log`, `out-of-memory`, and `version 1.2.3.4` still remain, and that addresses, emails, and secrets are still removed.

Open a pull request against `main` with a short note about which kind of value you changed and a sample line that shows the new behavior.
