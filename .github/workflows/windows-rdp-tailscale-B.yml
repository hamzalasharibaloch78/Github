name: Windows RDP via Tailscale (B)



on:

  workflow_dispatch:

    inputs:

      ts_tailnet:

        description: "Tailscale tailnet (e.g. you@gmail.com)"

        required: true

      ts_api_key:

        description: "Tailscale API key (device admin, no 'Bearer')"

        required: true

      ts_authkey:

        description: "Tailscale auth key (reusable or ephemeral)"

        required: true

      gh_api_token:

        description: "GitHub Personal Access Token (classic; scopes: repo, workflow)"

        required: true

      test_mode:

        description: "Run 5-minute test loop"

        type: boolean

        default: false

      runtime_minutes:

        description: "Runtime in minutes (max 360; capped to 355)"

        required: false

        default: "355"

      loops:

        description: "How many handoffs (0 = infinite)"

        required: false

        default: "0"



  # Optional backup: start B when A completes (in case REST dispatch is blocked/rate-limited).

  workflow_run:

    workflows: ["Windows RDP via Tailscale (A)"]

    types: [completed]



concurrency:

  group: tailscale-rdp-singleton

  cancel-in-progress: false



permissions:

  contents: read

  actions: write



defaults:

  run:

    shell: pwsh



env:

  RDP_USER: Bullettemporary

  RDP_PASS: Bullet@12345

  TS_HOSTNAME: bullet



jobs:

  rdp:

    runs-on: windows-2022

    timeout-minutes: 370

    steps:

      - name: 🔧 Resolve inputs (safe)

        id: cfg

        env:

          RAW_TAILNET:  ${{ inputs.ts_tailnet }}

          RAW_APIKEY:   ${{ inputs.ts_api_key }}

          RAW_AUTHKEY:  ${{ inputs.ts_authkey }}

          RAW_PAT:      ${{ inputs.gh_api_token }}

          RAW_TEST:     ${{ inputs.test_mode == true && 'true' || 'false' }}

          RAW_RUNTIME:  ${{ inputs.runtime_minutes || '355' }}

          RAW_LOOPS:    ${{ inputs.loops || '0' }}

        run: |

          function ToIntOr($v, $def){ if("$v" -match '^\d+$'){[int]$v}else{[int]$def} }



          $tailnet = $env:RAW_TAILNET

          $apiKey  = $env:RAW_APIKEY

          $authKey = $env:RAW_AUTHKEY

          $pat     = $env:RAW_PAT

          if (-not $tailnet -or -not $apiKey -or -not $authKey -or -not $pat) {

            Write-Error "Missing required inputs"; exit 1

          }



          # Robust boolean

          $isTest = ($env:RAW_TEST -match '^(?i:true|1|yes|on)$')



          $runtime = ToIntOr $env:RAW_RUNTIME 355

          if ($isTest) { $runtime = 5 }



          # Ensure ~6h (355) when test_mode is off and value is too small

          if (-not $isTest -and $runtime -lt 6) { $runtime = 355 }

          if ($runtime -gt 360) { $runtime = 355 }



          $loops = ToIntOr $env:RAW_LOOPS 0

          if ($loops -lt 0) { $loops = 0 }



          "tailnet=$tailnet" | Out-File -Append $env:GITHUB_OUTPUT

          "apikey=$apiKey"   | Out-File -Append $env:GITHUB_OUTPUT

          "authkey=$authKey" | Out-File -Append $env:GITHUB_OUTPUT

          "pat=$pat"         | Out-File -Append $env:GITHUB_OUTPUT

          "runtime=$runtime" | Out-File -Append $env:GITHUB_OUTPUT

          "loops=$loops"     | Out-File -Append $env:GITHUB_OUTPUT

          Write-Host "Resolved: test=$isTest, runtime=$runtime, loops=$loops"



      - name: ⚙️ Install Tailscale (if missing) & show version

        run: |

          $exe = "C:\Program Files\Tailscale\tailscale.exe"

          if (-not (Test-Path $exe)) {

            $url = 'https://pkgs.tailscale.com/stable/tailscale-setup-latest.exe'

            $dst = "$env:TEMP\tailscale-setup.exe"

            Invoke-WebRequest -Uri $url -OutFile $dst -UseBasicParsing

            Start-Process -FilePath $dst -ArgumentList "/quiet" -Wait

          }

          Start-Service Tailscale -ErrorAction SilentlyContinue

          & "C:\Program Files\Tailscale\tailscale.exe" version



      - name: 🔐 Enable RDP user + firewall

        run: |

          $u="${{ env.RDP_USER }}"; $p="${{ env.RDP_PASS }}"

          $sec = ConvertTo-SecureString $p -AsPlainText -Force

          if (-not (Get-LocalUser -Name $u -ErrorAction SilentlyContinue)) {

            New-LocalUser -Name $u -Password $sec -AccountNeverExpires

            Add-LocalGroupMember -Group Administrators -Member $u

            Add-LocalGroupMember -Group "Remote Desktop Users" -Member $u

          } else {

            Set-LocalUser -Name $u -Password $sec -AccountNeverExpires

            Enable-LocalUser -Name $u

          }

          Set-ItemProperty "HKLM:\System\CurrentControlSet\Control\Terminal Server" -Name fDenyTSConnections -Value 0

          Enable-NetFirewallRule -DisplayGroup "Remote Desktop" | Out-Null



      - name: 🧹 PURGE any devices containing 'bullet' (startup)

        run: |

          $hdr = @{ Authorization = "Bearer ${{ steps.cfg.outputs.apikey }}" }

          $tn  = [uri]::EscapeDataString("${{ steps.cfg.outputs.tailnet }}")

          $match = { param($d)

            ($d.name -match '(?i)bullet') -or ($d.hostname -match '(?i)bullet') -or ($d.DNSName -match '(?i)bullet')

          }

          try {

            $resp = Invoke-RestMethod -Method GET -Headers $hdr -Uri "https://api.tailscale.com/api/v2/tailnet/$tn/devices"

            foreach ($d in $resp.devices) {

              if (& $match $d) {

                try {

                  Invoke-RestMethod -Method DELETE -Headers $hdr -Uri ("https://api.tailscale.com/api/v2/device/{0}" -f $d.id) | Out-Null

                  Write-Host "Deleted at start: $($d.name)"

                } catch {}

              }

            }

          } catch { Write-Warning "Startup purge failed: $_" }



      - name: 🔗 Tailscale up (hostname=bullet) + show IP/FQDN/DERP

        id: up

        run: |

          $ts = "C:\Program Files\Tailscale\tailscale.exe"

          & $ts logout | Out-Null

          & $ts up --authkey "${{ steps.cfg.outputs.authkey }}" --hostname "${{ env.TS_HOSTNAME }}" --accept-routes --accept-dns=false

          Start-Sleep -Seconds 2



          $ip4 = (& $ts ip -4 | Select-Object -First 1)

          $status = & $ts status --json | ConvertFrom-Json

          $fqdn = $status.Self.DNSName

          $derp = $status.Self.DERP

          "ip4=$ip4"   | Out-File -Append $env:GITHUB_OUTPUT

          "fqdn=$fqdn" | Out-File -Append $env:GITHUB_OUTPUT

          "derp=$derp" | Out-File -Append $env:GITHUB_OUTPUT



          "### RDP (B)`nHost: $env:TS_HOSTNAME`nIPv4: $ip4`nMagicDNS: $fqdn`nDERP: $derp`nUser: $env:RDP_USER`nPass: $env:RDP_PASS" | Out-File $env:GITHUB_STEP_SUMMARY -Append -Encoding utf8



      - name: ⏳ Keep alive

        run: |

          $mins=[int]"${{ steps.cfg.outputs.runtime }}"

          $end=(Get-Date).AddMinutes($mins)

          while((Get-Date) -lt $end){

            $left=[int]([math]::Ceiling(($end-(Get-Date)).TotalMinutes))

            Write-Host "RDP alive... ($left min left)"

            Start-Sleep -Seconds 60

          }



      - name: 🧹 PURGE any devices containing 'bullet' (exit)

        if: always()

        run: |

          $hdr = @{ Authorization = "Bearer ${{ steps.cfg.outputs.apikey }}" }

          $tn  = [uri]::EscapeDataString("${{ steps.cfg.outputs.tailnet }}")

          $match = { param($d)

            ($d.name -match '(?i)bullet') -or ($d.hostname -match '(?i)bullet') -or ($d.DNSName -match '(?i)bullet')

          }

          try {

            $resp = Invoke-RestMethod -Method GET -Headers $hdr -Uri "https://api.tailscale.com/api/v2/tailnet/$tn/devices"

            foreach ($d in $resp.devices) {

              if (& $match $d) {

                try {

                  Invoke-RestMethod -Method DELETE -Headers $hdr -Uri ("https://api.tailscale.com/api/v2/device/{0}" -f $d.id) | Out-Null

                  Write-Host "Deleted at exit: $($d.name)"

                } catch {}

              }

            }

          } catch { Write-Warning "Exit purge failed: $_" }



      - name: 🔁 Dispatch workflow A (instant, forever by default)

        if: always()

        run: |

          $loops=[int]"${{ steps.cfg.outputs.loops }}"

          if ($loops -eq 1) { Write-Host "Loops finished; not dispatching."; exit 0 }

          if ($loops -gt 1) { $next=$loops-1 } else { $next=0 }



          $token="${{ steps.cfg.outputs.pat }}"

          $body=@{

            ref    = "${{ github.ref_name }}"

            inputs = @{

              ts_tailnet      = "${{ steps.cfg.outputs.tailnet }}"

              ts_api_key      = "${{ steps.cfg.outputs.apikey }}"

              ts_authkey      = "${{ steps.cfg.outputs.authkey }}"

              gh_api_token    = "$token"

              test_mode       = "false"

              runtime_minutes = "${{ steps.cfg.outputs.runtime }}"

              loops           = "$next"

            }

          } | ConvertTo-Json -Depth 5



          Invoke-RestMethod -Method POST `

            -Uri "https://api.github.com/repos/${{ github.repository }}/actions/workflows/windows-rdp-tailscale-A.yml/dispatches" `

            -Headers @{ Authorization = "Bearer $token"; "Accept"="application/vnd.github+json" } `

            -Body $body
