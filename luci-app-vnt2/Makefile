include $(TOPDIR)/rules.mk

LUCI_TITLE:=LuCI support for VNT2 VNTS2
LUCI_DEPENDS:=+luci-base +luci-compat +kmod-tun
LUCI_PKGARCH:=all

PKG_NAME:=luci-app-vnt2
PKG_VERSION:=1.7.4
PKG_RELEASE:=3
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
if [ "$${PKG_UPGRADE}" = "1" ] || [ -e "$${UPGRADE_MARKER}" ]; then
    rm -f "$${UPGRADE_MARKER}"
else
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
INITD=/etc/init.d
BACKUP_DIR=/tmp/vnt2-luci-app-backup
UPGRADE_MARKER=/tmp/vnt2-luci-app-upgrade
PURGE_LIST=/tmp/vnt2-luci-app-purge
CFG_DIR=$$(uci -q get vnt2.global.config_path)
WEB_DIR=$$(uci -q get vnt2.global.web_data_dir)
BIN_DIR=$$(uci -q get vnt2.global.bin_path)
CFG_DIR=$${CFG_DIR:-/etc/vnt2_config}
WEB_DIR=$${WEB_DIR:-/vnt_config}
BIN_DIR=$${BIN_DIR:-/usr/bin}
if [ "$$1" = "upgrade" ]; then
    rm -rf "$${BACKUP_DIR}"
    mkdir -p "$${BACKUP_DIR}/cfg" "$${BACKUP_DIR}/web"
    cp -f /etc/config/vnt2 "$${BACKUP_DIR}/vnt2" 2>/dev/null
    cp -a "$${CFG_DIR}/." "$${BACKUP_DIR}/cfg/" 2>/dev/null
    cp -a "$${WEB_DIR}/." "$${BACKUP_DIR}/web/" 2>/dev/null
    : > "$${UPGRADE_MARKER}"
    exit 0
fi
$${INITD}/vnt2 stop 2>/dev/null || true
$${INITD}/vnt2 fw_purge 2>/dev/null || true
sed -i '/vnt2_auto_update/d' /etc/crontabs/root 2>/dev/null
{
    for b in vnt2_web vnt2_cli vnt2_ctrl vnts2; do
        printf '%s\n' "$${BIN_DIR}/$${b}"
    done
    printf '%s\n' "$${CFG_DIR}" "$${WEB_DIR}" /etc/config/vnt2 /tmp/vnt2_log /var/run/vnt2 "$${BACKUP_DIR}"
} > "$${PURGE_LIST}"
rm -f "$${UPGRADE_MARKER}"
rm -rf /tmp/luci-*

exit 0
endef

define Package/$(PKG_NAME)/postrm
#!/bin/sh
[ -n "$${IPKG_INSTROOT}" ] && exit 0
PURGE_LIST=/tmp/vnt2-luci-app-purge
[ -f "$${PURGE_LIST}" ] || exit 0
tries=0
while [ -d /var/run/vnt2/fw_worker.lock ] && [ "$$tries" -lt 30 ]; do
    sleep 1
    tries=$$((tries + 1))
done
while IFS= read -r path; do
    case "$${path}" in
        /?*) rm -rf "$${path}" ;;
    esac
done < "$${PURGE_LIST}"
rm -f "$${PURGE_LIST}"
/etc/init.d/rpcd reload >/dev/null 2>&1 || true
rm -rf /tmp/luci-*

exit 0
endef

$(eval $(call BuildPackage,$(PKG_NAME)))
