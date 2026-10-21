include $(TOPDIR)/rules.mk

LUCI_TITLE:=LuCI support for VNT2 VNTS2
LUCI_DEPENDS:=+luci-base +luci-compat +kmod-tun
LUCI_PKGARCH:=all

PKG_NAME:=luci-app-vnt2
PKG_VERSION:=1.7.4
PKG_RELEASE:=5
PKG_PO_VERSION:=$(PKG_VERSION)

LUCI_MINIFY_JS:=0
LUCI_MINIFY_CSS:=0

PKG_LICENSE:=Apache-2.0
PKG_MAINTAINER:=whzhni1

include $(TOPDIR)/feeds/luci/luci.mk

define Build/Prepare/$(LUCI_NAME)
	$(SED) "s/^const ASSET_VERSION = '[^']*';/const ASSET_VERSION = '$(PKG_VERSION)';/" $(PKG_BUILD_DIR)/htdocs/luci-static/resources/vnt2/common.js
endef

define Package/$(PKG_NAME)/conffiles
/etc/config/vnt2
endef

define Package/$(PKG_NAME)/postinst
#!/bin/sh
[ -n "$${IPKG_INSTROOT}" ] && exit 0
INITD=/etc/init.d
BACKUP_DIR=/tmp/vnt2-luci-app-backup
UPGRADE_MARKER=/tmp/vnt2-luci-app-upgrade
PURGE_LIST=/tmp/vnt2-luci-app-purge
rm -f "$${PURGE_LIST}"
if [ -f /etc/uci-defaults/99-vnt2 ]; then
    /etc/uci-defaults/99-vnt2 || true
    rm -f /etc/uci-defaults/99-vnt2
fi
upgrade=0
if [ "$${PKG_UPGRADE}" = "1" ] || [ -e "$${UPGRADE_MARKER}" ] || pidof vnt2_web vnt2_cli vnts2 >/dev/null 2>&1; then
    upgrade=1
fi
rm -f "$${UPGRADE_MARKER}"
if [ "$$upgrade" = "0" ]; then
    if [ -d "$${BACKUP_DIR}" ]; then
        [ -f "$${BACKUP_DIR}/vnt2" ] && cp -f "$${BACKUP_DIR}/vnt2" /etc/config/vnt2
        CFG_DIR=$$(uci -q get vnt2.global.config_path)
        WEB_DIR=$$(uci -q get vnt2.global.web_data_dir)
        CFG_DIR=$${CFG_DIR:-/etc/vnt2_config}
        WEB_DIR=$${WEB_DIR:-/vnt_config}
        mkdir -p "$${CFG_DIR}" "$${WEB_DIR}"
        for f in "$${BACKUP_DIR}"/config/*; do
            [ -f "$$f" ] && cp -p "$$f" "$${CFG_DIR}/"
        done
        for f in "$${BACKUP_DIR}"/web/*; do
            [ -f "$$f" ] && cp -p "$$f" "$${WEB_DIR}/"
        done
        rm -rf "$${BACKUP_DIR}"
    fi
    as="$$(uci -q get vnt2.global.autostart)"
    if [ "$$as" != "0" ]; then
        $${INITD}/vnt2 enable
        ( setsid "$${INITD}/vnt2" start </dev/null >/dev/null 2>&1 & )
    fi
fi
$${INITD}/rpcd reload >/dev/null 2>&1 || true
rm -rf /tmp/luci-*

exit 0
endef

define Package/$(PKG_NAME)/prerm
#!/bin/sh
[ -n "$${IPKG_INSTROOT}" ] && exit 0
BACKUP_DIR=/tmp/vnt2-luci-app-backup
UPGRADE_MARKER=/tmp/vnt2-luci-app-upgrade
PURGE_LIST=/tmp/vnt2-luci-app-purge
pkg_upgrading() {
    local pid="$$$$" depth=0 args
    [ "$${PKG_UPGRADE}" = "1" ] && return 0
    case " $$* " in *" upgrade "*) return 0 ;; esac
    while [ "$$depth" -lt 16 ]; do
        case "$$pid" in ''|*[!0-9]*|0|1) return 1 ;; esac
        args="$$(tr '\0' ' ' <"/proc/$$pid/cmdline" 2>/dev/null)"
        case "$${args%% *}" in
            opkg|*/opkg) case " $$args " in *" install "*|*" upgrade "*) return 0 ;; esac ;;
            apk|*/apk) case " $$args " in *" add "*|*" upgrade "*|*" fix "*) return 0 ;; esac ;;
        esac
        pid="$$(sed -n 's/^PPid:[[:space:]]*//p' "/proc/$$pid/status" 2>/dev/null)"
        depth=$$((depth + 1))
    done
    return 1
}
rm -f "$${PURGE_LIST}"
if pkg_upgrading "$$@"; then
    : > "$${UPGRADE_MARKER}"
    exit 0
fi
rm -f "$${UPGRADE_MARKER}"
CFG_DIR=$$(uci -q get vnt2.global.config_path)
WEB_DIR=$$(uci -q get vnt2.global.web_data_dir)
BIN_DIR=$$(uci -q get vnt2.global.bin_path)
CFG_DIR=$${CFG_DIR:-/etc/vnt2_config}
WEB_DIR=$${WEB_DIR:-/vnt_config}
BIN_DIR=$${BIN_DIR:-/usr/bin}
rm -rf "$${BACKUP_DIR}"
mkdir -p "$${BACKUP_DIR}/config" "$${BACKUP_DIR}/web"
cp -f /etc/config/vnt2 "$${BACKUP_DIR}/vnt2" 2>/dev/null
for f in "$${CFG_DIR}"/*.vnt "$${CFG_DIR}"/*.vnts; do
    [ -f "$$f" ] && cp -p "$$f" "$${BACKUP_DIR}/config/"
done
for f in "$${WEB_DIR}"/*.toml; do
    [ -f "$$f" ] && cp -p "$$f" "$${BACKUP_DIR}/web/"
done
{
    for b in vnt2_web vnt2_cli vnt2_ctrl vnts2; do
        printf 'file %s\n' "$${BIN_DIR}/$${b}"
    done
    printf 'file %s\n' /etc/config/vnt2
    printf 'tree %s\n' /tmp/vnt2_log /var/run/vnt2
    printf 'glob %s\n' "$${CFG_DIR}/*.vnt" "$${CFG_DIR}/*.vnts" "$${WEB_DIR}/*.toml"
    printf 'dir %s\n' "$${CFG_DIR}" "$${WEB_DIR}"
} > "$${PURGE_LIST}"
exit 0
endef

define Package/$(PKG_NAME)/postrm
#!/bin/sh
[ -n "$${IPKG_INSTROOT}" ] && exit 0
[ -e /etc/init.d/vnt2 ] && exit 0
PURGE_LIST=/tmp/vnt2-luci-app-purge
[ -f "$${PURGE_LIST}" ] || exit 0
ubus call service delete '{"name":"vnt2"}' >/dev/null 2>&1 || true
for p in $$(pidof vnt2_web vnt2_cli vnt2_ctrl vnts2 2>/dev/null); do
    kill "$$p" 2>/dev/null
done
uci -q show firewall 2>/dev/null | sed -n 's/^firewall\.\(vnt2[^.=]*\)=.*/\1/p' | while read -r sid; do
    uci -q delete "firewall.$${sid}"
done
uci -q commit firewall
uci -q show network 2>/dev/null | sed -n 's/^network\.\(vnt2[^.=]*\)=interface$/\1/p' | while read -r sid; do
    uci -q delete "network.$${sid}"
done
uci -q commit network
sed -i '/vnt2_auto_update/d' /etc/crontabs/root 2>/dev/null
/etc/init.d/network reload >/dev/null 2>&1 || true
/etc/init.d/firewall reload >/dev/null 2>&1 || true
while read -r kind path; do
    case "$${kind}" in
        file) rm -f "$${path}" ;;
        tree) rm -rf "$${path}" ;;
        glob)
            for f in $${path}; do
                [ -f "$$f" ] && rm -f "$$f"
            done
            ;;
        dir) rmdir "$${path}" 2>/dev/null ;;
    esac
done < "$${PURGE_LIST}"
rm -f "$${PURGE_LIST}" /tmp/vnt2-luci-app-upgrade
/etc/init.d/rpcd reload >/dev/null 2>&1 || true
rm -rf /tmp/luci-*

exit 0
endef

$(eval $(call BuildPackage,$(PKG_NAME)))
