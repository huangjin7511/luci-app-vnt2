include $(TOPDIR)/rules.mk

LUCI_TITLE:=LuCI support for VNT2 VNTS2
LUCI_DEPENDS:=+luci-base +luci-compat +kmod-tun
LUCI_PKGARCH:=all

PKG_NAME:=luci-app-vnt2
PKG_VERSION:=1.7.4
PKG_RELEASE:=4
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
if [ -d "$${BACKUP_DIR}" ]; then
    [ -f "$${BACKUP_DIR}/vnt2" ] && cp -f "$${BACKUP_DIR}/vnt2" /etc/config/vnt2
    CFG_DIR=$$(uci -q get vnt2.global.config_path)
    WEB_DIR=$$(uci -q get vnt2.global.web_data_dir)
    CFG_DIR=$${CFG_DIR:-/etc/vnt2_config}
    WEB_DIR=$${WEB_DIR:-/vnt_config}
    mkdir -p "$${CFG_DIR}" "$${WEB_DIR}"
    cp -a "$${BACKUP_DIR}/cfg/." "$${CFG_DIR}/" 2>/dev/null
    cp -a "$${BACKUP_DIR}/web/." "$${WEB_DIR}/" 2>/dev/null
    rm -rf "$${BACKUP_DIR}"
fi
upgrade=0
if [ "$${PKG_UPGRADE}" = "1" ] || [ -e "$${UPGRADE_MARKER}" ] || pidof vnt2_web vnt2_cli vnts2 >/dev/null 2>&1; then
    upgrade=1
fi
rm -f "$${UPGRADE_MARKER}"
if [ "$$upgrade" = "0" ]; then
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
CFG_DIR=$$(uci -q get vnt2.global.config_path)
WEB_DIR=$$(uci -q get vnt2.global.web_data_dir)
BIN_DIR=$$(uci -q get vnt2.global.bin_path)
CFG_DIR=$${CFG_DIR:-/etc/vnt2_config}
WEB_DIR=$${WEB_DIR:-/vnt_config}
BIN_DIR=$${BIN_DIR:-/usr/bin}
rm -rf "$${BACKUP_DIR}"
mkdir -p "$${BACKUP_DIR}/cfg" "$${BACKUP_DIR}/web"
cp -f /etc/config/vnt2 "$${BACKUP_DIR}/vnt2" 2>/dev/null
cp -a "$${CFG_DIR}/." "$${BACKUP_DIR}/cfg/" 2>/dev/null
cp -a "$${WEB_DIR}/." "$${BACKUP_DIR}/web/" 2>/dev/null
: > "$${UPGRADE_MARKER}"
{
    for b in vnt2_web vnt2_cli vnt2_ctrl vnts2; do
        printf '%s\n' "$${BIN_DIR}/$${b}"
    done
    printf '%s\n' "$${CFG_DIR}" "$${WEB_DIR}" /etc/config/vnt2 /tmp/vnt2_log /var/run/vnt2 "$${BACKUP_DIR}" "$${UPGRADE_MARKER}" "$${PURGE_LIST}"
} > "$${PURGE_LIST}"
exit 0
endef

define Package/$(PKG_NAME)/postrm
#!/bin/sh
[ -n "$${IPKG_INSTROOT}" ] && exit 0
[ -e /etc/init.d/vnt2 ] && exit 0
PURGE_LIST=/tmp/vnt2-luci-app-purge
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
if [ -f "$${PURGE_LIST}" ]; then
    while IFS= read -r path; do
        case "$${path}" in
            /?*) rm -rf "$${path}" ;;
        esac
    done < "$${PURGE_LIST}"
fi
rm -f "$${PURGE_LIST}"
/etc/init.d/rpcd reload >/dev/null 2>&1 || true
rm -rf /tmp/luci-*

exit 0
endef

$(eval $(call BuildPackage,$(PKG_NAME)))
