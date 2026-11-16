#!/data/data/com.termux/files/usr/bin/bash
# SillyTavern 一鍵包 - Android (Termux) 安裝腳本
set -e

c() { printf '\033[%sm%s\033[0m\n' "$1" "$2"; }
say()  { c 36 "$*"; }
ok()   { c 32 "  [OK] $*"; }
warn() { c 33 "  [!]  $*"; }
fail() { c 31 "  [X]  $*"; exit 1; }

if [ -z "$PREFIX" ] || [ ! -d "$PREFIX" ]; then
  fail "請在 Termux 裡執行這個腳本（從 F-Droid 或 GitHub 安裝的 Termux，不要用 Play 商店舊版）。"
fi

ST_DIR="$HOME/SillyTavern"

c 35 "============================================="
c 35 "     SillyTavern 酒館 一鍵安裝（Termux）"
c 35 "============================================="
echo
echo "選擇下載來源："
echo "  [1] 官方來源（台灣／海外，預設）"
echo "  [2] 中國大陸鏡像（Termux 套件走清華、npm 走 npmmirror）"
printf "輸入 1 或 2 後按 Enter："
read -r CHOICE </dev/tty || CHOICE=1

if [ "$CHOICE" = "2" ]; then
  say "切換 Termux 套件來源到清華鏡像…"
  sed -i 's@^\(deb.*stable main\)$@#\1\ndeb https://mirrors.tuna.tsinghua.edu.cn/termux/apt/termux-main stable main@' "$PREFIX/etc/apt/sources.list" || warn "切換鏡像失敗，繼續用原本來源"
fi

echo
say "[1/3] 更新 Termux 並安裝 Git、Node.js（會跑一段時間）"
yes | pkg update -y -o Dpkg::Options::="--force-confnew" || warn "pkg update 有警告，繼續安裝"
yes | pkg upgrade -y -o Dpkg::Options::="--force-confnew" || warn "pkg upgrade 有警告，繼續安裝"
pkg install -y git nodejs-lts || fail "安裝 Git / Node.js 失敗，請檢查網路後重跑。"
ok "Node.js $(node -v)、$(git --version)"

if [ "$CHOICE" = "2" ]; then
  npm config set registry https://registry.npmmirror.com
  ok "npm 已改用 npmmirror"
fi

echo
say "[2/3] 下載酒館"
if [ -f "$ST_DIR/server.js" ]; then
  ok "酒館已存在，略過下載"
else
  rm -rf "$ST_DIR"
  git clone -b release --depth 1 https://github.com/SillyTavern/SillyTavern.git "$ST_DIR" \
    || fail "下載酒館失敗（GitHub 連不上？）請檢查網路後重跑。"
  ok "酒館下載完成"
fi

echo
say "[3/3] 安裝酒館需要的套件（第一次會久一點）"
cd "$ST_DIR"
npm install --no-audit --no-fund --loglevel=error --no-progress --omit=dev \
  || fail "npm install 失敗，請檢查網路後重跑。"
ok "套件安裝完成"

# 建立快捷指令：st = 啟動，st-update = 更新
cat > "$PREFIX/bin/st" <<'EOF'
#!/data/data/com.termux/files/usr/bin/bash
cd "$HOME/SillyTavern" || { echo "找不到酒館，請重新執行安裝腳本"; exit 1; }
command -v termux-wake-lock >/dev/null && termux-wake-lock
echo "酒館啟動中… 用手機瀏覽器打開 http://127.0.0.1:8000"
echo "（要關閉：在這裡按 Ctrl + C）"
./start.sh
EOF

cat > "$PREFIX/bin/st-update" <<'EOF'
#!/data/data/com.termux/files/usr/bin/bash
set -e
cd "$HOME/SillyTavern" || { echo "找不到酒館"; exit 1; }
STAMP=$(date +%Y%m%d-%H%M)
mkdir -p "$HOME/st-backups"
if [ -d data ]; then
  tar -czf "$HOME/st-backups/data-$STAMP.tar.gz" data config.yaml 2>/dev/null || tar -czf "$HOME/st-backups/data-$STAMP.tar.gz" data
  echo "[OK] 已備份資料到 ~/st-backups/data-$STAMP.tar.gz"
fi
git pull --ff-only
npm install --no-audit --no-fund --loglevel=error --no-progress --omit=dev
echo "[OK] 更新完成！輸入 st 啟動"
EOF

chmod +x "$PREFIX/bin/st" "$PREFIX/bin/st-update"

echo
c 32 "============================================="
c 32 "  安裝完成！"
c 32 "  之後打開 Termux 輸入  st         就能啟動"
c 32 "           輸入  st-update  就能更新"
c 32 "============================================="
echo
printf "現在就啟動酒館嗎？(Y/n)："
read -r GO </dev/tty || GO=n
case "$GO" in
  n|N) ;;
  *) exec st ;;
esac
