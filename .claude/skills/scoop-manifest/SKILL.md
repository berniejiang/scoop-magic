---
name: scoop-manifest
description: |
  Generate and update Scoop bucket manifest JSON files for Windows software packages.
  Use this skill whenever the user asks to create, write, update, or fix a Scoop manifest,
  or when they mention "scoop manifest", "bucket", "scoop installer", "app manifest",
  or want to add a new app to their Scoop bucket. Also use when the user provides a
  download URL, GitHub repo, or installer file and wants a scoop-compatible JSON manifest
  for it. Works with all installer types: InnoSetup, NSIS, MSI-as-zip, 7z/zip portable,
  and custom installer scripts.
---

# Scoop Manifest Generator

You are helping maintain a Scoop bucket. Each JSON file in the bucket root is a manifest that tells Scoop how to download, install, and update a Windows application.

## When generating a manifest

1. Ask the user for the app's name, homepage/download URL, and any specifics about how the installer works. If they provide a GitHub repo, extract everything you can from the releases page.
2. Determine the installer type from the download file extension and packaging method.
3. Generate the complete JSON manifest following the patterns below.
4. Save it as `<appname>.json` in the bucket root directory.

## Manifest Structure

Every manifest is a single JSON object. Fields should appear in this conventional order:

```json
{
    "version": "...",
    "description": "...",
    "homepage": "...",
    "license": "...",
    (architecture or url+hash),
    (extract_dir / extract_to),
    (innosetup),
    (installer / uninstaller),
    (pre_install / post_install),
    (bin),
    (shortcuts),
    (persist),
    (notes),
    "checkver": {...},
    "autoupdate": {...}
}
```

## Required Fields

- **version**: The app's version string (e.g. `"1.2.3"`, `"2024-01-15"`)
- **description**: One-line description. Don't include the app name if it matches the filename.
- **homepage**: URL to the app's website or GitHub repo.
- **license**: SPDX identifier string (e.g. `"GPL-3.0-or-later"`, `"Apache-2.0"`), or an object with `identifier` and optional `url`. Common values:
  - `"Freeware"` — free to use, closed source
  - `"Proprietary"` — paid / commercial
  - `"Public Domain"`, `"Shareware"`, `"Unknown"`
  - Object form: `{"identifier": "Freeware", "url": "https://example.com/eula"}`

## URL & Hash

Two patterns depending on whether the app has architecture-specific builds:

**Single architecture** — flat `url` and `hash`:
```json
"url": "https://example.com/app-1.0.zip",
"hash": "sha256:abcdef..."
```

**Multiple architectures** — nested under `architecture`:
```json
"architecture": {
    "64bit": {
        "url": "https://example.com/app-1.0-x64.zip",
        "hash": "sha256:..."
    },
    "32bit": {
        "url": "https://example.com/app-1.0-x86.zip",
        "hash": "sha256:..."
    },
    "arm64": {
        "url": "https://example.com/app-1.0-arm64.zip",
        "hash": "sha256:..."
    }
}
```

Hash format: plain hex string (SHA256 assumed), or prefixed: `"md5:..."`, `"sha512:..."`, `"sha1:..."`.

## Installer Types — When to Use Each

### Type 1: Portable Archive (zip / 7z / tar.xz)

The simplest case. The downloaded archive extracts to a runnable app — no installer executable.

**When:** Download is a .zip, .7z, .tar.xz, .tar.gz that contains the app files directly.

**Key fields:**
- `extract_dir`: Subdirectory inside the archive to extract (if the archive contains a wrapper folder).
- `bin`: Executable(s) to add to PATH. String or array.
- `shortcuts`: `[[exe, "Start Menu Name"], ...]`

**Example — single arch portable zip:**
```json
{
    "version": "3.1.4.0",
    "homepage": "https://example.com/",
    "description": "A screenshot tool",
    "license": "Freeware",
    "architecture": {
        "64bit": {
            "url": "https://dl.example.com/app-3.1.4.0.zip",
            "hash": "abcdef..."
        }
    },
    "extract_dir": "AppName",
    "bin": "app.exe",
    "shortcuts": [["app.exe", "App Name"]],
    "persist": ["Config", "Data"],
    "checkver": {
        "url": "https://example.com/download",
        "regex": "app-([\\d.]+)\\.zip"
    },
    "autoupdate": {
        "architecture": {
            "64bit": {
                "url": "https://dl.example.com/app-$version.zip"
            }
        }
    }
}
```

**Example — multi-arch 7z with bin aliases:**
```json
{
    "version": "2026-03-22",
    "description": "A software distro for Windows",
    "homepage": "https://www.msys2.org/",
    "license": "GPL-2.0-only|BSD-3-Clause",
    "architecture": {
        "64bit": {
            "url": "https://mirrors.tuna.tsinghua.edu.cn/msys2/distrib/x86_64/msys2-base-x86_64-20260322.tar.xz",
            "hash": "..."
        }
    },
    "extract_dir": "msys64",
    "bin": [
        ["msys2_shell.cmd", "msys2", "-msys2 -defterm -here -no-start"],
        ["msys2_shell.cmd", "mingw", "-mingw -defterm -here -full-path -no-start"]
    ],
    "shortcuts": [["msys2.exe", "MSYS2"]],
    "persist": "home",
    "checkver": {
        "url": "https://github.com/msys2/msys2-installer/releases",
        "regex": "/releases/tag/(?<version>\\d{4}-\\d{2}-\\d{2})"
    },
    "autoupdate": {
        "architecture": {
            "64bit": {
                "url": "https://mirrors.tuna.tsinghua.edu.cn/msys2/distrib/x86_64/msys2-base-x86_64-$version.tar.xz"
            }
        }
    }
}
```

### Type 2: InnoSetup

InnoSetup-based `.exe` installers can be extracted by Scoop automatically.

**When:** The download is a `.exe` that was built with InnoSetup (common for many Windows apps). You can usually tell by checking if running `7z l installer.exe` shows `{app}` entries, or if the app's download page mentions InnoSetup.

**Key fields:**
- `"innosetup": true` — tells Scoop this is an InnoSetup installer so it can extract files without running the installer.
- No `installer` section needed — Scoop handles extraction automatically.
- No `#/dl.7z` URL fragment needed — Scoop recognizes InnoSetup natively.

**Example:**
```json
{
    "version": "3.2.135",
    "description": "EDA Professional",
    "homepage": "https://pro.lceda.cn/",
    "license": "Proprietary",
    "innosetup": true,
    "architecture": {
        "64bit": {
            "url": "https://image.lceda.cn/files/lceda-pro-windows-x64-3.2.135.exe",
            "hash": "..."
        },
        "arm64": {
            "url": "https://image.lceda.cn/files/lceda-pro-windows-arm64-3.2.135.exe",
            "hash": "..."
        }
    },
    "shortcuts": [["lceda-pro.exe", "LCEDA Pro"]],
    "checkver": {
        "url": "https://lceda.cn/page/download",
        "regex": "/lceda-pro-windows-x64-([\\d.]+).exe"
    },
    "autoupdate": {
        "architecture": {
            "64bit": {
                "url": "https://image.lceda.cn/files/lceda-pro-windows-x64-$version.exe"
            },
            "arm64": {
                "url": "https://image.lceda.cn/files/lceda-pro-windows-arm64-$version.exe"
            }
        }
    }
}
```

### Type 3: NSIS / Electron / Custom EXE — Re-packaged as 7z

Many apps (Electron apps, NSIS installers, etc.) ship as `.exe` installers that we don't want to run directly. Instead, append `#/dl.7z` to the URL so Scoop downloads it and extracts with 7-Zip.

**When:** The download is an `.exe` installer (NSIS, InstallShield, Electron builder, etc.) that is NOT InnoSetup. Also used when the installer is a self-extracting archive or when you want to bypass the installer GUI.

**Key technique — URL fragment `#/dl.7z`:**
```
"https://example.com/app-setup-1.0.exe#/dl.7z"
```
Scoops downloads `app-setup-1.0.exe` but saves it as `dl.7z`, then extracts it with 7-Zip.

**Common post_install cleanup after NSIS extraction:**
```json
"post_install": "Remove-Item \"$dir\\`$PLUGINSDIR\" -Force -Recurse"
```

**Example — NSIS app with 7z extraction:**
```json
{
    "version": "3.0.7",
    "homepage": "https://pot-app.com/",
    "description": "A cross-platform translation software",
    "license": "GPL-3.0-only",
    "architecture": {
        "64bit": {
            "url": "https://github.com/pot-app/pot-desktop/releases/download/3.0.7/pot_3.0.7_x64-setup.exe#/dl.7z",
            "hash": "..."
        },
        "32bit": {
            "url": "https://github.com/pot-app/pot-desktop/releases/download/3.0.7/pot_3.0.7_x86-setup.exe#/dl.7z",
            "hash": "..."
        }
    },
    "post_install": "Remove-Item \"$dir\\`$PLUGINSDIR\" -Force -Recurse",
    "shortcuts": [["pot.exe", "Pot"]],
    "persist": "config.toml",
    "checkver": {
        "github": "https://github.com/pot-app/pot-desktop"
    },
    "autoupdate": {
        "architecture": {
            "64bit": {
                "url": "https://github.com/pot-app/pot-desktop/releases/download/$version/pot_$version_x64-setup.exe#/dl.7z"
            },
            "32bit": {
                "url": "https://github.com/pot-app/pot-desktop/releases/download/$version/pot_$version_x86-setup.exe#/dl.7z"
            }
        }
    }
}
```

**NSIS/Electron `$PLUGINSDIR` pattern:**

Many Electron apps packaged with NSIS contain the actual app in `$PLUGINSDIR/app-64.7z` (or `app-32.7z`). After 7z extraction of the outer `.exe`, you need to further extract this inner archive:

```json
"architecture": {
    "64bit": {
        "url": "https://example.com/app-1.0-setup.exe#/dl.7z",
        "hash": "...",
        "installer": {
            "script": [
                "Expand-7ZipArchive \"$dir\\`$PLUGINSDIR\\app-64.7z\" \"$dir\" -Removal",
                "Remove-Item \"$dir\\`$PLUGINSDIR\",\"$dir\\`$R0\" -Force -Recurse"
            ]
        }
    }
}
```

Or equivalently using `pre_install` (when installer script is not inside architecture):
```json
"pre_install": [
    "Expand-7zipArchive \"$dir\\`$PLUGINSDIR\\app-64.7z\" \"$dir\" -Removal",
    "Remove-Item \"$dir\\`$PLUGINSDIR\",\"$dir\\`$R0\" -Force -Recurse"
]
```

**Example — complex NSIS with extract_to:**
```json
{
    "version": "7.68.6",
    "homepage": "https://feishu.cn/",
    "description": "AI era productivity platform",
    "license": {"identifier": "EULA", "url": "https://feishu.cn/en/terms"},
    "architecture": {
        "64bit": {
            "url": "https://sf3-cn.feishucdn.com/obj/ee-appcenter/abc/Feishu-win32_x64-7.68.6-signed.exe#/dl.7z",
            "hash": "md5:..."
        },
        "32bit": {
            "url": "https://sf3-cn.feishucdn.com/obj/ee-appcenter/def/Feishu-win32_ia32-7.68.6-signed.exe#/dl.7z",
            "hash": "md5:..."
        }
    },
    "extract_to": "app",
    "shortcuts": [["app/Feishu.exe", "Feishu"]],
    "checkver": {...},
    "autoupdate": {...}
}
```

### Type 4: MSI

MSI files should be treated like archives — extract files from the MSI without running the installer. Simply omit the deprecated `msi` property. Often the URL ends in `.msi` but we rename it via URL fragment.

**When:** The download is an `.msi` file.

**Key technique:** Append `#.msi` or just use the `.msi` URL directly — Scoop handles MSI extraction natively.

**Example:**
```json
{
    "version": "1.44.55.0",
    "homepage": "https://getquicker.net/",
    "description": "A quick launcher tool",
    "license": "Freeware",
    "architecture": {
        "64bit": {
            "url": "https://getquicker.net/download/item/fast_x64#.msi",
            "hash": "..."
        },
        "32bit": {
            "url": "https://getquicker.net/download/item/fast_x86#.msi",
            "hash": "..."
        }
    },
    "extract_dir": "Quicker",
    "bin": "Quicker.exe",
    "shortcuts": [["Quicker.exe", "Quicker"]],
    "checkver": {
        "url": "https://getquicker.net/Download",
        "regex": "version[\\w\\W]*?(\\d.[\\d.]+)"
    },
    "autoupdate": {
        "architecture": {
            "64bit": {
                "url": "https://getquicker.net/download/item/fast_x64#.msi"
            },
            "32bit": {
                "url": "https://getquicker.net/download/item/fast_x86#.msi"
            }
        }
    }
}
```

### Type 5: Custom Script Installers

When the installer requires custom PowerShell logic (registry manipulation, multi-step extraction, external runtime data linking, etc.), use `installer`/`uninstaller` with `script`.

**When:** The app needs special handling beyond simple extraction — registry entries, protocol handlers, complex file moves, or external data directory linking.

**Example — registry manipulation:**
```json
"installer": {
    "script": [
        "if (!(is_admin)) { error 'Admin required'; exit 1 }",
        "New-Item Registry::HKEY_CLASSES_ROOT\\myapp -Force | Out-Null",
        "New-ItemProperty Registry::HKEY_CLASSES_ROOT\\myapp -Name 'URL Protocol' -Value '' -Force | Out-Null"
    ]
},
"uninstaller": {
    "script": [
        "if (!(is_admin)) { error 'Admin required'; exit 1 }",
        "Remove-Item Registry::HKEY_CLASSES_ROOT\\myapp -Force -Recurse -ErrorAction SilentlyContinue"
    ]
}
```

**Example — multi-step 7z extraction (complex NSIS/InstallShield):**
```json
"pre_install": [
    "Expand-7zipArchive \"$dir\\$fname\"",
    "Expand-7zipArchive \"$dir\\`$_11_\\`$EXEFILE\" -Switches '-t#'",
    "Move-Item \"$dir\\`$_11_\\*.7z\" \"$dir\"",
    "Remove-Item \"$dir\\*\" -Exclude '4.7z', '2.7z' -Recurse",
    "Expand-7zipArchive \"$dir\\2.7z\" -ExtractDir 'CONTROL\\office6' -Removal",
    "Expand-7zipArchive \"$dir\\4.7z\" -ExtractDir 'office6' -Removal"
]
```

**Example — external runtime data linking (for apps that store config in AppData):**
```json
"installer": {
    "script": [
        "Import-Module $(Join-Path $(Find-BucketDirectory -Root -Name extras-cn) scripts/AppsUtils.psm1)",
        "Mount-ExternalRuntimeData -Source \"$persist_dir\\appdata\" -Target \"$env:APPDATA\\appname\"",
        "Remove-Module -Name AppsUtils"
    ]
},
"uninstaller": {
    "script": [
        "Import-Module $(Join-Path $(Find-BucketDirectory -Root -Name extras-cn) scripts/AppsUtils.psm1)",
        "Dismount-ExternalRuntimeData -Target \"$env:APPDATA\\appname\"",
        "Remove-Module -Name AppsUtils"
    ]
}
```

## Common Optional Fields

### bin

Add executables to the user's PATH. Supports aliases and arguments:
```json
"bin": "app.exe"
"bin": ["app.exe", "tool.exe"]
"bin": [["app.exe", "alias", "--flag"]]
```
When using alias form for a single entry, wrap in outer array: `"bin": [["app.exe", "alias"]]`

### shortcuts

Start menu shortcuts. Each entry is `[exe_path, shortcut_name, optional_args, optional_icon]`:
```json
"shortcuts": [["app.exe", "App Name"]]
"shortcuts": [["bin/app.exe", "App", "--flag", "icon.ico"]]
```
Supports subdirectories: `"SubDir\\App Name"`

### persist

Files/directories to persist across updates. Uses a junction from install dir to persist dir:
```json
"persist": "config.ini"
"persist": ["config", "data", ["original_name", "alias_name"]]
```

### pre_install / post_install

PowerShell scripts run before/after installation. Available variables: `$dir`, `$persist_dir`, `$version`, `$original_dir`.

### notes

Displayed to the user after installation:
```json
"notes": "Run 'app --setup' to complete configuration."
"notes": ["Note 1", "Note 2"]
```

### env_add_path / env_set

```json
"env_add_path": "bin",
"env_set": {"MYAPP_HOME": "$dir"}
```

## checkver — Version Detection

### GitHub releases (simplest):
```json
"checkver": "github"
```
Requires `homepage` to be the GitHub repo URL. Matches `releases/tag/(v)?X.Y.Z`.

```json
"checkver": {"github": "https://github.com/user/repo"}
```

### Regex on a webpage:
```json
"checkver": {
    "url": "https://example.com/download",
    "regex": "app-([\\d.]+)\\.exe"
}
```

### JSON API with JSONPath:
```json
"checkver": {
    "url": "https://api.example.com/version.json",
    "jsonpath": "$.latestVersion"
}
```

### JSON API with both JSONPath and regex:
```json
"checkver": {
    "url": "https://api.github.com/repos/user/repo/releases",
    "jp": "$[0].assets[*].browser_download_url",
    "regex": "/releases/download/v([\\d.]+)/"
}
```

### PowerShell script for complex scenarios:
```json
"checkver": {
    "url": "https://example.com/api/downloads",
    "script": ["$data = $page | ConvertFrom-Json", "$data.version"],
    "regex": "([\\d.]+)"
}
```

### Named capture groups (for use in autoupdate):
```json
"checkver": {
    "url": "https://example.com/downloads",
    "regex": "file_(?<build>\\d+)_v(?<version>[\\d.]+)"
}
```
Captured groups become `$matchBuild`, `$matchVersion` in autoupdate.

## autoupdate — Automatic Update Rules

The `autoupdate` section tells `checkver.ps1` how to generate a new manifest when a new version is found.

### Version variables available in URL templates:
- `$version` — full version (e.g. `3.7.1`)
- `$majorVersion`, `$minorVersion`, `$patchVersion`, `$buildVersion` — split on `.`
- `$underscoreVersion` — `3_7_1`
- `$dashVersion` — `3-7-1`
- `$cleanVersion` — `371` (digits only)
- `$matchX` / `$matchName` — from checkver capture groups

### URL variables available in hash section:
- `$url` — the autoupdate URL without fragments
- `$baseurl` — URL without filename
- `$basename` — filename from URL (ignores `#/...` fragment)

### Basic autoupdate (GitHub releases):
```json
"autoupdate": {
    "architecture": {
        "64bit": {
            "url": "https://github.com/user/repo/releases/download/v$version/app-$version-win64.zip"
        },
        "32bit": {
            "url": "https://github.com/user/repo/releases/download/v$version/app-$version-win32.zip"
        }
    }
}
```

### With hash extraction from a text file:
```json
"autoupdate": {
    "url": "https://example.com/app-$version.zip",
    "hash": {
        "url": "$baseurl/app-$version.sha256"
    }
}
```

### With hash from GitHub release page:
```json
"autoupdate": {
    "architecture": {
        "64bit": {
            "url": "https://github.com/user/repo/releases/download/$version/app-$version-win64.zip"
        }
    },
    "hash": {
        "url": "https://github.com/user/repo/releases/$version",
        "regex": "(?sm)$basename.*?$sha256"
    }
}
```

### With hash from JSON API:
```json
"autoupdate": {
    "architecture": {
        "64bit": {
            "url": "https://example.com/app-$version-x64.exe#/dl.7z",
            "hash": {
                "url": "https://example.com/api/downloads",
                "jsonpath": "$.versions.win64.hash"
            }
        }
    }
}
```

### Hash modes:
| Mode | Source | Key field |
|------|--------|-----------|
| `extract` (default) | Plain text / webpage | `regex` or `find` |
| `json` | JSON file / API | `jsonpath` or `jp` |
| `xpath` | XML file | `xpath` |
| `rdf` | RDF file | (automatic) |
| `metalink` | Metalink header/.meta4 | (automatic) |
| `fosshub` | FossHub URLs | (automatic) |
| `sourceforge` | SourceForge URLs | (automatic) |
| `download` | Download & hash locally | (fallback) |

## Decision Flowchart

When generating a manifest, follow this decision process:

1. **Is the download a portable archive** (`.zip`, `.7z`, `.tar.xz`, `.tar.gz`)?
   → Use **Type 1** (Portable Archive). Set `extract_dir` if needed.

2. **Is the download an InnoSetup `.exe`?**
   → Use **Type 2** (InnoSetup). Set `"innosetup": true`.

3. **Is the download an `.exe` installer** (NSIS, Electron builder, etc.)?
   → Use **Type 3** (Re-package as 7z). Append `#/dl.7z` to URL. Add `post_install` to clean up `$PLUGINSDIR`.

4. **Is the download an `.msi` file?**
   → Use **Type 4** (MSI). Scoop extracts MSI natively, no `msi` property needed.

5. **Does the app need special handling** (registry, complex extraction, protocol handlers)?
   → Use **Type 5** (Custom Script). Write `installer`/`uninstaller` scripts.

## Chinese Mirror Patterns

This bucket uses Chinese mirrors for many downloads. Common patterns:

- GitHub releases via TUNA mirror: `https://mirrors.tuna.tsinghua.edu.cn/github-release/user/repo/LatestRelease/file.zip`
- VLC via USTC mirror: `https://mirrors.ustc.edu.cn/videolan-ftp/vlc/`
- Blender via TUNA mirror: `https://mirrors.tuna.tsinghua.edu.cn/blender/release/`
- MSYS2 via TUNA mirror: `https://mirrors.tuna.tsinghua.edu.cn/msys2/distrib/`

When a Chinese mirror is available, prefer it for faster downloads in China.

## Checklist Before Saving

Verify the generated manifest:
- [ ] `version` matches the actual latest version
- [ ] `url` is a valid, accessible download link
- [ ] `hash` is correct (SHA256 by default, or specify algorithm prefix)
- [ ] `checkver` can find the version automatically
- [ ] `autoupdate` can generate new URLs when version changes
- [ ] `shortcuts` point to existing executables
- [ ] `bin` entries are correct (with aliases if needed)
- [ ] `persist` covers user data that should survive updates
- [ ] Installer type is correctly identified
- [ ] `post_install` cleans up NSIS/InnoSetup artifacts if needed
- [ ] JSON is valid (no trailing commas, proper escaping)
