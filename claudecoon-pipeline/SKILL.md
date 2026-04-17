# SKILL: ClaudeCoon Pipeline Setup

  ## What is ClaudeCoon
  Self-evolving algorithmic trading pipeline for NinjaTrader 8 on MES/MNQ futures.
  Runs on Replit (Python Flask, port 9000) + VPS (NT8).

  ## Architecture
  ```
  Replit (Flask pipeline, port 9000)
      ↓ TCP via ngrok
  VPS (NinjaTrader 8 ATI, port 36973)
      ↓ executes trades
  Live account (MES futures, account 1598925)
  ```

  ## Key environment variables
  | Variable | Value |
  |---|---|
  | NT_ATI_HOST | 7.tcp.ngrok.io |
  | NT_ATI_PORT | 23018 |
  | ANTHROPIC_API_KEY | your key |
  | GH_PASSWORD | GitHub PAT |
  | PIPELINE_API_KEY | claudecoon-live-2026 |

  ## VPS Setup
  1. Install ngrok to C:\\ngrok\\
  2. Run: ngrok tcp --remote-addr=7.tcp.ngrok.io:23018 36973
  3. Run launcher: python C:\\claudecoon\\claudecoon_launcher.py
  4. Open NT8 → load ClaudeCoonTimeFit strategy
  5. Set account to live account (1598925)
  6. Enable strategy

  ## Entry window
  09:30 - 10:14 ET weekdays (MES 5-minute chart)

  ## Automated schedule
  | Time UTC | Job |
  |---|---|
  | 00:30 | Mini GA evolution |
  | 02:00 | Morning briefing |
  | 06:00 | Strategy discovery |
  | 09:00 | Regime detection |
  | 09:15 | Pre-market deploy |
  | 22:00 | TimeFit daily refit |
  | Sat 00:00 | Deep GA (50 generations) |
  | Sun 00:00 | Full pipeline run |

  ## Test ATI connection
  ```python
  import socket
  s = socket.create_connection(("7.tcp.ngrok.io", 23018), timeout=10)
  s.sendall(b"CONNECTED\n")
  resp = s.recv(1024)
  print(resp.decode())
  # Expected: 2Orders|2Strategies|11172RealizedPnL|...Sim1012
  s.close()
  ```

  ## Common issues
  | Issue | Fix |
  |---|---|
  | ATI disconnected | Check ngrok is running on VPS |
  | Wrong tunnel address | Update NT_ATI_HOST/PORT in Replit Secrets |
  | Strategy not trading | Check Enabled checkbox in NT8 Strategies tab |
  | No trades in entry window | Check EMA cross + RSI entry conditions |
  | pipeline_state.json missing | State served live at http://localhost:9000/status |
  | git pull blocked in agent | Use GitHub REST API with GH_PASSWORD PAT |
  