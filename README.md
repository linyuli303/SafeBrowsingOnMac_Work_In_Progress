**BRAVE**

sudo find / -xdev \( -iname "*brave*" -o -iname "*BraveSoftware*" \) 2>/dev/null | grep -v "^/System"
pgrep -fl -i brave

**ZEN**

sudo find /System/Volumes/Data -xdev \( -iname "zen" -o -iname "zen.*" -o -iname "zen-*" -o -iname "zen_*" -o -iname "zen *" -o -iname "*zen-browser*" -o -iname "*zenbrowser*" \) 2>/dev/null | grep -v "^/System/Volumes/Data/Applications/Zen.app/" | sort
