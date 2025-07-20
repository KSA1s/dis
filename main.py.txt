import os
import time
import json
import string
import random
import datetime
import requests
from colorama import Fore, init
init(autoreset=True)

# Global config from environment
MULTI_TOKEN = os.getenv("MULTI_TOKEN", "false").lower() == "true"
TOKEN = os.getenv("DISCORD_TOKEN")
TOKENS = os.getenv("TOKENS", "").splitlines()
WEBHOOK_URL = os.getenv("WEBHOOK_URL", "")
DELAY = float(os.getenv("DEFAULT_DELAY", "1.5"))
SAMPLE_PUNCTUATION = "_."
AVAILABLE_FILE = "available_usernames.txt"
USERNAME_LIST_FILE = "usernames.txt"
URL = "https://discord.com/api/v9/users/@me/pomelo-attempt"
SYS_URL = "https://discord.com/api/v9/users/@me"
integ_0 = 0
available_usernames = []

Lb = Fore.LIGHTBLACK_EX
Ly = Fore.LIGHTYELLOW_EX

def get_token():
    if MULTI_TOKEN:
        return TOKENS[integ_0]
    return TOKEN

def s_sys_h():
    return {
        "Content-Type": "application/json",
        "Origin": "https://discord.com/",
        "Authorization": get_token()
    }

def get_account_info():
    try:
        response = requests.get(SYS_URL, headers=s_sys_h())
        if response.status_code == 200:
            user = response.json()
            return f"{user['username']}#{user['discriminator']}"
        return "Unknown#0000"
    except:
        return "Error#0000"

def save(username):
    with open(AVAILABLE_FILE, "a") as f:
        f.write(f"{username}\n")

def send_webhook(username):
    if not WEBHOOK_URL:
        return
    embed = {
        "title": f"Username `{username}` is available ✅",
        "timestamp": str(datetime.datetime.utcnow()),
        "color": 16768000,
        "footer": {"text": "DSV - Discord Username Checker"},
        "author": {
            "name": "DSV",
            "url": "https://github.com/suenerve/Discord-Username-Checker"
        }
    }
    data = {
        "username": "DSV",
        "avatar_url": "https://cdn.icon-icons.com/icons2/488/PNG/512/search_47686.png",
        "embeds": [embed]
    }
    try:
        requests.post(WEBHOOK_URL, json=data)
    except Exception as e:
        print(f"{Lb}[!]{Fore.RED} Webhook failed: {e}")

def validate(username):
    global integ_0
    body = {"username": username}
    while True:
        response = requests.post(URL, headers=s_sys_h(), json=body)
        if response.status_code == 429:
            if MULTI_TOKEN:
                print(f"{Lb}[!]{Fore.YELLOW} Token {integ_0} is rate limited. Switching...")
                integ_0 = (integ_0 + 1) % len(TOKENS)
                continue
            else:
                retry = response.json().get("retry_after", 5)
                print(f"{Lb}[!]{Fore.RED} Rate limited. Sleeping for {retry}s...")
                time.sleep(retry)
                continue
        break

    data = response.json()
    if "taken" in data:
        if not data["taken"]:
            print(f"{Lb}[+]{Fore.LIGHTGREEN_EX} '{username}' is available.")
            available_usernames.append(username)
            save(username)
            send_webhook(username)
        else:
            print(f"{Lb}[-]{Fore.RED} '{username}' is taken.")
    else:
        print(f"{Lb}[?]{Fore.RED} Error: {data.get('message', 'Unknown error')}")

def run_checker():
    print(f"{Fore.CYAN}Logged in as {get_account_info()}")
    print(f"{Fore.YELLOW}Reading usernames from {USERNAME_LIST_FILE}...")

    if not os.path.exists(USERNAME_LIST_FILE):
        print(f"{Fore.RED}Error: '{USERNAME_LIST_FILE}' not found.")
        return

    with open(USERNAME_LIST_FILE, "r") as f:
        usernames = [line.strip() for line in f if line.strip()]

    for username in usernames:
        validate(username)
        time.sleep(DELAY)

    print(f"{Fore.GREEN}Finished. Found {len(available_usernames)} available usernames.")
    print(f"{Fore.GREEN}Saved to '{AVAILABLE_FILE}'.")

if __name__ == "__main__":
    run_checker()
