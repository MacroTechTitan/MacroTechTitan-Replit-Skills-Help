# SKILL: ngrok Static TCP Tunnel

  ## Problem
  ngrok changes address on every restart.
  This breaks any service that connects via ngrok.

  ## Solution — Reserved TCP Address

  ### Step 1 — Get reserved address
  1. Go to dashboard.ngrok.com
  2. Left sidebar → Universal Gateway → TCP Addresses
  3. Click New TCP Address
  4. Copy the address (e.g. 7.tcp.ngrok.io:23018)

  ### Step 2 — Start ngrok with reserved address
  On Windows VPS:
  ```
  C:\\ngrok\\ngrok.exe tcp --remote-addr=7.tcp.ngrok.io:23018 36973
  ```

  On Mac/Linux:
  ```
  ngrok tcp --remote-addr=7.tcp.ngrok.io:23018 36973
  ```

  ### Step 3 — Add auto-restart to launcher
  ```python
  import subprocess, requests, time

  NGROK_RESERVED = "7.tcp.ngrok.io:23018"
  LOCAL_PORT = 36973

  def check_ngrok_alive():
      try:
          resp = requests.get("http://127.0.0.1:4040/api/tunnels", timeout=5)
          return len(resp.json().get("tunnels", [])) > 0
      except:
          return False

  def restart_ngrok():
      print("[Launcher] ngrok down — restarting...")
      subprocess.run(["taskkill", "/f", "/im", "ngrok.exe"], capture_output=True)
      time.sleep(2)
      subprocess.Popen([r"C:\\ngrok\\ngrok.exe", "tcp",
                        f"--remote-addr={NGROK_RESERVED}", str(LOCAL_PORT)])
      time.sleep(5)
      print("[Launcher] ngrok restarted")

  while True:
      if not check_ngrok_alive():
          restart_ngrok()
      time.sleep(60)
  ```

  ### Step 4 — Save address to Replit Secrets
  | Secret | Value |
  |---|---|
  | NT_ATI_HOST | 7.tcp.ngrok.io |
  | NT_ATI_PORT | 23018 |

  These never change with a reserved address.

  ### Cost
  Requires ngrok Pay-as-you-go or higher ($20/month).
  Worth it — eliminates all tunnel address issues.
  