## Wake up alarm on Aztec Validator downtime: For IOS

I wanted to make an alarm that would wake me up during the night if my node went down or started missing attestations. 
UptimeRobot has a paid option to make a call, but a call isn't enough especially if sleep mode or "do not disturb" is on, so through this method the phone will trigger an alarm at max volume and bypass sleep or "do not disturb" mode.

---

## What this setup intends to accomplish
1. Detect if your Validator is unresponsive or is missing attestations.
2. Triggers an annoying high volume alarm which bypasses sleep and "do not disturb" mode.
3. You deel with it.

---

## ✅ 1. Install OpenResty

```bash
sudo apt-get update
sudo apt-get install -y curl gnupg ca-certificates lsb-release

curl -s https://openresty.org/package/pubkey.gpg | sudo gpg --dearmor -o /usr/share/keyrings/openresty.gpg

echo "deb [signed-by=/usr/share/keyrings/openresty.gpg] http://openresty.org/package/debian $(lsb_release -sc) openresty" | \
sudo tee /etc/apt/sources.list.d/openresty.list

sudo apt-get update
sudo apt-get install -y openresty openresty-opm
```

---

## 📦 2. Install Lua HTTP client (resty.http)

```bash
sudo /usr/local/openresty/bin/opm install ledgetech/lua-resty-http
```

---

## 🧹 3. Create config include file

```bash
sudo nano /usr/local/openresty/nginx/conf/healthcheck.conf
```

Paste:


> ⚠️ **Warning**: If you're not part of the validator set yet use this script instead: [healthcheck.conf](https://github.com/joaolago1113/wake-up-aztec-node/blob/main/healthcheck.conf), otherwise use this:

> ⚠️ **Warning**: Replace <YOUR_VALIDATOR_ADDRESS> with your address.
```nginx
lua_shared_dict health_cache 1m;
limit_req_zone $binary_remote_addr zone=health_limit:10m rate=5r/s;

server {
    listen 8089;
    server_name _;

    location /health {
        limit_req zone=health_limit burst=10 nodelay;

        if ($request_method != GET) {
            return 405;
        }

        if ($http_user_agent = "") {
            return 403;
        }

        content_by_lua_block {
            local addr = ("<YOUR_VALIDATOR_ADDRESS>"):lower()

            local http = require "resty.http"
            local cjson = require "cjson"

            -- Check cache
            local cache = ngx.shared.health_cache
            local cached_response = cache:get("health_result")
            if cached_response then
                ngx.say(cached_response)
                return
            end

            local httpc = http.new()
            httpc:set_timeout(30000) -- 30 second

            local res, err = httpc:request_uri("http://127.0.0.1:8080", {
                method = "POST",
                body = '{"method": "node_getValidatorsStats"}',
                headers = { ["Content-Type"] = "application/json" }
            })

            if not res then
                ngx.status = 502
                ngx.say("backend error")
                return
            end

            local ok, data = pcall(cjson.decode, res.body)
            if not ok or not data.result or not data.result.stats then
                ngx.status = 500
                ngx.say("invalid backend response")
                return
            end

            local stats = data.result.stats[addr]
            if not stats or not stats.missedAttestations then
                ngx.status = 500
                ngx.say("data missing")
                return
            end

            local rate = stats.missedAttestations.rate
            if rate == nil then
                local count = stats.missedAttestations.count
                if count == nil or count ~= 0 then
                    ngx.status = 500
                    ngx.say("data missing")
                    return
                else
                    rate = 0
                end
            end

            local result

            if rate > 0.8 then
                ngx.status = 503
                result = "rate too high: " .. rate
            else
                ngx.status = 200
                result = "OK"
            end

            cache:set("health_result", result, 59)  -- cache for 5 seconds
            ngx.say(result)
        }
    }
}
```

---

## 🧷 4. Include config in OpenResty main config

Edit the main config:

```bash
sudo nano /usr/local/openresty/nginx/conf/nginx.conf
```

Inside the `http { ... }` block, **add**:

```nginx
include healthcheck.conf;
```

---

## 🚫 5. Stop and disable default Nginx

If you have nginx running, either change it's default port from 80 to something else, or stop it:

```bash
sudo systemctl stop nginx
sudo systemctl disable nginx
```

---

## 🚀 6. Enable and start OpenResty

```bash
sudo systemctl enable openresty
sudo systemctl start openresty
```

---

## 🚫 7. Open port 8089 on your firewall

##### e.g. if using `ufw`:

```
sudo ufw allow 8089
```

---

## 🧪 8. Test health endpoint

Local test:

```bash
curl http://127.0.0.1:8089/health
```

Remote test:

```bash
curl http://<your-server-ip>:8089/health
```

Both should return "OK"

---

You're now running an OpenResty health check endpoint.

---

## 📲 9. Set Up Monitoring with UptimeRobot

1. Go to [https://uptimerobot.com](https://uptimerobot.com) and create an account.
2. **Important**: Use your iCloud email (ending in `@icloud.com`) when signing up.
   - This is required later to use the iOS Mail app for Shortcut-based notification automation.

Make sure the email used in UptimeRobot matches the one configured in your iPhone's Mail app.

---

## 🛠️ 10. Add a Monitor in UptimeRobot

1. Log into your [UptimeRobot dashboard](https://uptimerobot.com/dashboard) app.
2. Click on the **+** at the top right.
3. Choose "Keyword".
4. Fill in the fields as follows:
   - For "URL or IP address" fill in `http://<your-server-ip>:8089/health`
   - > ⚠️ **Warning**: Replace `<your-server-ip>` with your validator's ip.
   - For the "Monitor name" fill in: `Aztec 1` *(or Aztec 2, Aztec 3, etc., if you are monitoring multiple nodes)*
   - For Keyword write `OK`and choose "be present" below.
5. Click **Create Monitor**.
6. Press on the 3 vertical dots at the top right and choose "Manage aler contacts"
7. Switch on the "Email contact" option.

#### It will look like this

<img src="https://github.com/user-attachments/assets/9414d8f2-eb60-4ded-9a2e-b22b7f54adde" width="400" />

---

## 🚨 11. Automate Notifications on iOS

1. Get the iOS Shortcut:
   [https://www.icloud.com/shortcuts/6a45d796f86846aab583490cdb4549a5](https://www.icloud.com/shortcuts/6a45d796f86846aab583490cdb4549a5)
2. Add a music, or an alarm sound to the file section of the shortcut:
   > ⚠️ **Note**: You can use this alarm sound [Annoying_Alarm_Clock-UncleKornicob-420925725.wav.zip](https://github.com/user-attachments/files/20509587/Annoying_Alarm_Clock-UncleKornicob-420925725.wav.zip)
   <img src="https://github.com/user-attachments/assets/f95ff327-d2b0-4c8a-8f2e-163445a656d7" width="400" />
3. Open the **Shortcuts** app on your iPhone.
4. Go to the **Automation** tab.
5. Tap **+** at the top right.
6. Scroll and select **Email**.
7. In the field **Subject Contains**, choose and write: `Monitor is DOWN: Aztec`
8. Select **Run Immediately**.
9. Tap **Next**.
10. Under **My Shortcuts**, choose the shortcut you got in step **0**: **Aztec Alarm 1**.

#### It will look like this
<img src="https://github.com/user-attachments/assets/3d2e33ae-380f-4007-994c-9228dea747ae" width="400" />

## Concluding

Now when UptimeRobot detects downtime or missin attestations, your iPhone will trigger a shortcut-based alarm upon UpTimeRobot sending you an email to your `@icloud` Mail app with a subject containing "Monitor DOWN: Aztec...".

> ⚠️ **Warning**: To stop the shortcut alarm once it has gone off: Open **Siri** by long pressing the power button.

