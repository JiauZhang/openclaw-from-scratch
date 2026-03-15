# Node

```shell
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.4/install.sh | bash
source ~/.bashrc
nvm install node
npm install -g pnpm@latest
pnpm self-update
```

# OpenClaw

```shell
git clone git@github.com:openclaw/openclaw.git --depth 1
cd openclaw

pnpm install
pnpm ui:build # auto-installs UI deps on first run
pnpm build

pnpm openclaw onboard --install-daemon

# start dev loop (auto-reload on TS changes)
pnpm gateway:watch

# open in browser
http://127.0.0.1:18789

# stop gateway service
pnpm openclaw gateway stop
```
