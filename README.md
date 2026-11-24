**BRAVE**
sudo find / -xdev \( -iname "*brave*" -o -iname "*BraveSoftware*" \) 2>/dev/null | grep -v "^/System"
pgrep -fl -i brave

**ZEN**
sudo find / -xdev \( -iname "zen" -o -iname "zen.*" -o -iname "zen-*" -o -iname "zen_*" -o -iname "zen *" -o -iname "*zen-browser*" -o -iname "*zenbrowser*" -o -iname "app.zen*" \) 2>/dev/null | grep -v "^/System" | grep -v "^/Applications/Zen.app/" | sort
