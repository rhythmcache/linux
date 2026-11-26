# Mainline Linux Porting Recon Report: Poco X4 Pro 5G (`veux`, SM6375)

- **Target Device:** Xiaomi Poco X4 Pro 5G (`veux`, board ID `S88107BA1`, `androidboot.hwc=India`)
- **SoC:** Qualcomm Snapdragon 695 5G (`SM6375`, vendor codename `holi` / `blair`)
- **Vendor Kernel:** `5.4.302~Vector/e9f9d785` (`ThisColdBlueSky/VectorKernel`, `arch/arm64/boot/dts/vendor/xiaomi/veux.dts`)
- **Mainline Baseline:** `rhythmcache/linux` (`arch/arm64/boot/dts/qcom/sm6375.dtsi`, `sm6375-sony-xperia-murray-pdx225.dts`)

---

## 1. Live Device Telemetry Summary

All telemetry was collected read-only from the running device (`~/port/recon/`):

- **Kernel & Config (`/proc/config.gz`, `uname -a`, `lsmod`):**
  - `CONFIG_MODULES` is disabled (`# CONFIG_MODULES is not set`); `/proc/modules` does not exist, so all vendor drivers are built-in.
  - `CONFIG_DEBUG_FS=y`, though `/sys/kernel/debug` is mounted empty on this user/KSU build (no `clk_summary`, `regulator_summary`, `gpio`, or `pinctrl` debugfs entries exposed). Regulator states/voltages were captured from `/sys/class/regulator/regulator.*`.
- **Kernel Cmdline (`/proc/cmdline`):**
  - `androidboot.bootdevice=4804000.ufshc`
  - `androidboot.usbcontroller=4e00000.dwc3`
  - `msm_drm.dsi_display0=qcom,mdss_dsi_k6s_38_0c_0a_fhdp_dsc_vid:`
  - `androidboot.slot_suffix=_b`
- **Active I2C / SPI / SPMI / Bus Devices (`/sys/bus/*/devices`, `/proc/interrupts`):**
  - **I2C-3 (`4c90000.i2c`, QUPv3 SE10):** `3-0028` (`qcom,sn-nci` NFC), `3-0034`/`3-0035` (`awinic,aw882xx_smartpa`), `3-0036`/`3-0037` (`foursemi,fs19xx`), `3-004e` (`richtek,rt1711h` TCPC), `3-0051` (`lionsemi,ln8000`), `3-0066` (`sc,sc8551-master` actively probed in `dmesg`), `3-006a` (`ti,bq2589x-1` actively probed in `dmesg`).
  - **I2C-4 (`4a80000.i2c`, QUPv3 SE0):** `4-0051` (`lionsemi,ln8000_2`), `4-0055` (`bq27z561` fuel gauge), `4-005a` (`awinic,aw8624_haptic`, IRQ on `msmgpio 3`), `4-0066` (`sc,sc8551-slave`).
  - **I2C-5 (`4c88000.i2c`, QUPv3 SE8):** `5-005d` (`goodix,gt9896` touchscreen).
  - **SPI-0 (`4c84000.spi`, QUPv3 SE7):** `spi0.0` (`ir-spi-led`, `/dev/lirc0`).
  - **SPMI (`1c40000.qcom,spmi`):** `spmi0-00` & `spmi0-01` (`pm6125`), `spmi0-04` (`pmr735a`), `spmi0-06` (`pmk8350`).
  - **Fingerprint (`msmgpio 17`):** `soc:fpc1020` has 5144 interrupts in `/proc/interrupts` (FPC1020 is the active sensor on this unit, though `goodix_fp` and `silead_fp` nodes also exist in `veux-fingerprint.dtsi`).
  - **Bluetooth:** `ttyHS0` + `msmgpio 64` (`hs_uart_wakeup`) + `btfmslim_slave` on `a5c0000.slim`.
  - **SoundWire:** `swr0` (`va-macro`), `swr1` (`rx-macro`), `wcd937x-slave`.
- **Storage (`/dev/block/bootdevice/by-name`):** UFS (`sda`–`sdf`) with A/B slots (`boot_a/b`, `dtbo_a/b`, `vendor_boot_a/b`, `modem_a/b`, `dsp_a/b`, `bluetooth_a/b`, `super`, `userdata`, `persist`, `fsc`, `fsg`, `modemst1/2`).

---

## 2. Driver Gap Analysis

Classification categories:
- **Upstream (as-is):** Driver and bindings exist in mainline `~/port/mainline`; only a DT node in `sm6375-xiaomi-veux.dts` (or `sm6375.dtsi`) is needed.
- **Upstream (needs patches/bindings):** Driver exists in mainline, but needs SM6375/device-specific compatible additions, DT binding changes, `sm6375.dtsi` wiring, or minor quirks/patches.
- **Not upstream (needs porting):** No mainline driver; must be ported from `~/port/vendor-5.4` (vendor source paths listed).
- **Blob-only / Userspace-driven:** Driven by DSP/modem firmware or Android userspace HALs/daemons over generic IPC/character interfaces.

| Subsystem | Hardware / Vendor Node | Vendor Compatible | Mainline Status | Classification | Notes / Mainline or Vendor File |
|---|---|---|---|---|---|
| **Clocks** | GCC, GPUCC, RPMCC | `qcom,blair-gcc`, `qcom,blair-gpucc`, `qcom,rpmcc-holi` | `gcc-sm6375.c`, `gpucc-sm6375.c`, `clk-smd-rpm.c` in `sm6375.dtsi` | **Upstream, works as is** | Use `qcom,sm6375-gcc`, `qcom,sm6375-gpucc`, `qcom,rpmcc-sm6375`. |
| **Clocks (Display)** | DISPCC (`5f00000`) | `qcom,blair-dispcc` | `drivers/clk/qcom/dispcc-sm6375.c` | **Upstream, needs patches/bindings** | Driver is upstream (`qcom,sm6375-dispcc`), but node is not yet in `sm6375.dtsi`. |
| **Clocks (Camera)** | CAMCC | `qcom,blair-camcc` | No `camcc-sm6375.c` in mainline | **Not upstream, needs porting** | Vendor: `drivers/clk/qcom/camcc-blair.c` (or `holi`). |
| **Power / RPM** | SMD-RPM & RPMPD (note: SM6375 uses SMD-RPM, not RPMh) | `qcom,glink-rpm`, `qcom,sm6375-rpmpd` | `smd-rpm.c`, `rpmpd.c` in `sm6375.dtsi` | **Upstream, works as is** | Already wired in `sm6375.dtsi`. |
| **Regulators (PMIC)** | PM6125, PMR735A RPM regulators | `qcom,rpm-pm6125-regulators`, `qcom,rpm-pmr735a-regulators` | `qcom_smd-regulator.c`, `pm6125.dtsi`, `pmr735a.dtsi` | **Upstream, works as is** | Only board voltage limits needed in `veux` DTS. |
| **Pinctrl** | SoC TLMM (`400000`) & PMIC GPIOs | `qcom,blair-pinctrl`, `qcom,pm6125-gpio`, `qcom,pmr735a-gpio`, `qcom,pmk8350-gpio` | `pinctrl-sm6375.c` (`qcom,sm6375-tlmm`), `pinctrl-spmi-gpio.c` | **Upstream, works as is** | Already in `sm6375.dtsi` & PMIC `.dtsi`s. |
| **UFS** | `4804000.ufshc` + `4807000.ufsphy_mem` | `qcom,ufshc`, `qcom,ufs-phy-qmp-v4-yupik` | `ufs-qcom.c`, `phy-qcom-qmp-ufs.c` (no `sm6375` entry yet) | **Upstream, needs patches/bindings** | Needs `sm6375` UFS PHY init table/compatible in `phy-qcom-qmp-ufs.c` (similar to `sc7280`) + DT nodes in `sm6375.dtsi`. |
| **SD Card** | `4784000.sdhci` (`sdhc_2`, CD GPIO 94) | `qcom,sdhci-msm-v5` | `sdhci-msm.c` (`qcom,sm6375-sdhci`) in `sm6375.dtsi` | **Upstream, works as is** | Wire `vmmc` (`pm6125_l22`), `vqmmc` (`pm6125_l5`), `cd-gpios = <&tlmm 94 GPIO_ACTIVE_HIGH>`. |
| **USB / DWC3 / PHY** | `4ef8800` / `4e00000.dwc3` + `162b000.hsphy` | `qcom,dwc-usb3-msm`, `snps,dwc3`, `qcom,usb-hsphy-snps-femto` | `dwc3-qcom.c`, `phy-qcom-snps-femto-v2.c` in `sm6375.dtsi` | **Upstream, works as is** | USB2 works out of the box in `sm6375.dtsi` (`qcom,select-utmi-as-pipe-clk`); `veux` is USB 2.0. |
| **USB Type-C / PD** | `rt1711h@4e` (`3-004e`) + `rt-pd-manager` | `richtek,rt1711h`, `richtek,rt-pd-manager` | `drivers/usb/typec/tcpm/tcpci_rt1711h.c` | **Upstream, needs patches/bindings** | Use upstream `tcpci_rt1711h` + `connector` node for role/PD; drop `rt-pd-manager` vendor glue. |
| **Display (MDSS/DSI)** | `5e00000` MDSS, DPU 6.9, DSI0 + 7nm PHY | `qcom,mdss-dsi-ctrl`, `qcom,dsi-phy-7nm` | `msm_mdss.c`, `dpu_6_9_sm6375.h`, `dsi_phy.c` (`qcom,sm6375-dsi-phy-7nm`) | **Upstream, needs patches/bindings** | Drivers exist in mainline; MDSS/DPU/DSI DT nodes must be added to `sm6375.dtsi`. |
| **Display Panel** | Xiaomi `k6s_38_0c_0a` (1080x2400@120Hz DSC video OLED) | `qcom,mdss_dsi_k6s_38_0c_0a_fhdp_dsc_vid` | No upstream panel driver | **Not upstream, needs porting** | Needs new `drm_panel` driver converted from `veux-display.dtsi` + `dsi-panel-k6s-38-0c-0a-fhdp-dsc-video.dtsi` (DSC 1.1, 8bpp, 2 slices/pkt). `simple-framebuffer` at `0x85200000` works for early boot. |
| **GPU (Adreno 619)** | `5900000` Adreno 619 + `5940000` SMMU | `qcom,kgsl-3d0` | `a6xx_catalog.c` (`qcom,adreno-619.0`), `adreno_smmu` in `sm6375.dtsi` | **Upstream, needs patches/bindings** | Add `gpu@5900000` DT node to `sm6375.dtsi`; needs `a630_sqe.fw` + `a615_zap.*` firmware. |
| **Touchscreen** | Goodix GT9896 (`5-005d`, IRQ GPIO 22, RST 21, AVDD 51, DVDD 44) | `goodix,gt9896` | `goodix_berlin_*` supports GT9897/GT9916; `goodix.c` does not support GTx8 | **Not upstream, needs porting** | Vendor: `drivers/input/touchscreen/goodix_gtx8/` (`goodix_ts_i2c.c`, etc.). Needs `goodix_firmware.bin` / `goodix_cfg_group.bin`. |
| **PMICs** | PM6125 (`0,1`), PMR735A (`4`), PMK8350 (`6`) | `qcom,pm6125`, `qcom,pmr735a`, `qcom,pmk8350` | `pm6125.dtsi`, `pmr735a.dtsi`, `pmk8350.dtsi` (PON, RTC, ADC5/7, ADC-TM5, GPIO) | **Upstream, works as is** | All three PMICs are supported upstream. |
| **Charger (Main)** | TI BQ2589x (`3-006a`, IRQ GPIO 99, OTG GPIO 98) | `ti,bq2589x-1` | `drivers/power/supply/bq25890_charger.c` | **Upstream, needs patches/bindings** | Adapt DT properties to mainline `ti,bq25890` binding (drop `PSY_IIO` vendor glue). |
| **Fuel Gauge** | TI BQ27Z561 (`4-0055`) | `bq27z561` | `drivers/power/supply/bq27xxx_battery_i2c.c` (`ti,bq27z561`) | **Upstream, needs patches/bindings** | Change compatible to `ti,bq27z561`. |
| **Charge Pumps & Glue** | SC8551 (`3-0066`, `4-0066`), LN8000 (`3-0051`, `4-0051`), `wt_chg`, `pd_adapter` | `sc,sc8551-master/slave`, `lionsemi,ln8000`, `qcom,wt_chg`, `xiaomi,pd_adapter` | None | **Not upstream, needs porting** | Vendor: `drivers/power/supply/qcom/{sc8551_charger.c,ln8000_charger.c,wt_chg.c}`. Optional for basic 5V/9V charging via `bq25890`. |
| **Audio (SoC & Codec)** | Bolero (RX/VA macros), WCD937x (SoundWire), ADSP APR | `qcom,bolero-codec`, `qcom,wcd937x-codec` | `lpass-rx-macro.c`, `lpass-va-macro.c`, `wcd937x.c`, `wcd937x-sdw.c` | **Upstream, needs patches/bindings** | Needs SM6375 LPASS/Bolero/SoundWire DT wiring & machine driver (`sm8250.c` or `sc7280.c`) + UCM2 topology. |
| **Audio (Smart PA)** | Awinic AW882xx (`3-0034`, `3-0035`) / FourSemi FS19xx (`3-0036`, `3-0037`) | `awinic,aw882xx_smartpa`, `foursemi,fs19xx` | `aw88261.c` / `aw88395/` exist, not generic `aw882xx` | **Not upstream, needs porting** | Vendor: `techpack/audio/asoc/codecs/aw882xx/aw882xx.c`, `fs19xx/fs19xx.c`. Needs `aw882xx_pid_2113_acf.bin` / `fs1958.fsm`. |
| **Camera** | CCI0, CSIPHY, TFE/OPE, 4x sensors/EEPROM/actuator/flash | `qcom,cam-sensor`, `qcom,eeprom`, `qcom,cci` | `camss.c` has no `sm6375` support; no `camcc-sm6375` | **Not upstream, needs porting** | Vendor: `techpack/camera/drivers/`. High complexity (TFE/OPE ISP architecture). |
| **Sensors** | ADSP SSC (`adsprpc-smd` / `sscrpcd`) | `qcom,msm-fastrpc-compute` | `qcom_fastrpc.c` (upstream) + userspace `libssc` / `iio-sensor-proxy` | **Blob-only / userspace-driven** | Sensors sit on ADSP QUPs; accessed via FastRPC (`/dev/fastrpc-*`) + ADSP firmware + `/mnt/vendor/persist/sensors`. |
| **Wi-Fi** | WCN3990/WCN399x (`c800000`) | `qcom,icnss` (vendor `qca_cld3`) | `ath10k_snoc.c` (`qcom,wcn3990-wifi`) in `sm6375.dtsi` | **Upstream, works as is** | Mainline `ath10k_snoc` replaces `icnss2` + `qca_cld3`; needs `wlanmdsp.mbn` + `bdwlan.bin`. |
| **Bluetooth** | WCN3990 UART (`ttyHS0`, `4a84000`, wakeup GPIO 64) + SLIMbus | `qcom,wcn3990`, `qcom,btfmslim_slave` | `hci_qca.c` (`qcom,wcn3990-bt`) | **Upstream, needs patches/bindings** | Add `bluetooth` child node under `uart1` (`4a84000`) with PM6125/PMR735A supplies; `btfm_slim` (`drivers/bluetooth/btfm_slim.c`) is not upstream. |
| **Modem / Remoteproc / Video** | MPSS (`6080000`), ADSP (`a400000`), CDSP (`b300000`), IPA (`5800000`), Venus (`5ab0000`) | `qcom,pil-tz-generic`, `qcom,ipa`, `qcom,vidc` | `qcom_q6v5_pas.c` (in `sm6375.dtsi`); IPA/Venus lack `sm6375` support | **Upstream (PAS) / Needs patches (IPA, Venus)** | Remoteprocs work as-is with `modem.mdt`, `adsp.mdt`, `cdsp.mdt` + `rmtfs`. `venus` & `ipa` need SM6375 data/DT. |
| **Thermal** | TSENS0/1 (`4410000`, `4412000`), PMIC temp-alarm & ADC-TM5 | `qcom,tsens-v2`, `qcom,spmi-temp-alarm`, `qcom,spmi-adc-tm5` | In `sm6375.dtsi` & PMIC `.dtsi`s | **Upstream, works as is** | Already wired in `sm6375.dtsi`. |
| **Haptics** | Awinic AW8624 (`4-005a`, RST GPIO 2, IRQ GPIO 3) | `awinic,aw8624_haptic` | No mainline driver | **Not upstream, needs porting** | Vendor: `drivers/misc/haptic_nv/aw8624.c`. Needs `aw8624_*.bin` firmware. |
| **NFC** | NXP SN1xx (`3-0028`, IRQ 9, VEN 6, PNVen 86, FIRM 8, CLKREQ 7) | `qcom,sn-nci` | `drivers/nfc/nxp-nci/i2c.c` (`nxp,nxp-nci-i2c`) | **Upstream, needs patches/bindings** | Vendor: `drivers/nfc/qti/`. Mainline `nxp-nci` can be adapted or `sn-nci` ported. |
| **Consumer IR** | SPI IR LED (`4c84000.spi`, `spi0.0`) | `ir-spi-led` | `drivers/media/rc/ir-spi.c` (`ir-spi-led` in mainline?) | **Not upstream, needs porting** | Vendor: `drivers/media/rc/ir-spi.c` (mainline has `ir-spi.c`? Check compatible `ir-spi-led` vs `ir-spi`). |
| **Fingerprint** | FPC1020 (IRQ 17, RST 18); DT also lists Goodix/Silead | `fpc,fpc1020` | Proprietary TEE-driven | **Blob-only / userspace-driven** | Vendor: `drivers/input/fingerprint/fpc/fpc1020_platform_tee.c`. Requires QSEE TZ app (`fpctzapp64.mdt`). |
| **GPIO Keys** | Volume Down (`pmr735a_gpios 1`), Power/VolUp (`pon`) | `gpio-keys`, `qcom,pm8941-pwrkey/resin` | `gpio_keys.c`, `pm8941-pwrkey.c` | **Upstream, works as is** | Standard mainline DT nodes. |

---

## 3. Vendor HAL & Firmware Requirements

Scanned from `/vendor/etc/init/*.rc`, `/vendor/lib64/hw/`, `/vendor/bin/hw/`, `/vendor/ueventd.rc`, `/vendor/firmware`, and `/vendor/firmware_mnt/image/`.

### HALs -> Kernel Nodes / Sysfs / Firmware Blobs

| HAL / Daemon (`init/*.rc` & `bin/hw` / `lib64/hw`) | Kernel `/dev` & `/sys` Paths Opened | Required Firmware Blobs |
|---|---|---|
| **Display Composer** (`vendor.qti.hardware.display.composer-service`, `gralloc`, `mapper@4.0`) | `/dev/dri/card0`, `/dev/dri/renderD128`, `/dev/ion` (or `/dev/dma_heap`), `/dev/vndbinder`, `/sys/class/backlight/panel0-backlight/*` | `a615_zap.{mdt,b00-b02}`, `a630_sqe.fw` (GPU/Mesa) |
| **Health / Charger** (`android.hardware.health-service.qti`, `batterysecret`) | `/sys/class/power_supply/*`, `/sys/class/Charging_Adapter/pd_adapter/*`, `/sys/class/qcom-battery/odm_battery/*`, `/dev/dri`, `/dev/graphics/fb0`, `/dev/input` | None |
| **USB / Gadget** (`android.hardware.usb-service.qti`, `usb.gadget`) | `/sys/class/typec/port0/*`, `/sys/class/udc/*`, `/sys/class/power_supply/usb/*`, `/sys/class/qcom-battery/*`, `/sys/bus/platform/drivers/{msm-dwc3,xhci-hcd}/*`, `/sys/bus/iio/devices/iio:device4/*`, `/config/usb_gadget/*` | None |
| **Thermal** (`android.hardware.thermal-service.qti`) | `/sys/class/thermal/thermal_zone*/*`, `/sys/class/thermal/cooling_device*/*`, `/sys/devices/soc0/{soc_id,hw_platform}` | None |
| **Audio** (`android.hardware.audio.service`, `audio.primary.holi.so`) | `/dev/snd/*` (`pcmC0D*`, `comprC0D*`, `controlC0`, `hwC*D*`), `/dev/aw882xx_smartpa`, `/sys/class/thermal/thermal_zone*/type` | `adsp.{mdt,b*}`, `adsp2.{mdt,b*}`, `aw882xx_pid_2113_acf.bin` / `fs1958.fsm` |
| **Camera** (`android.hardware.camera.provider@2.4-service_64`, `camera.qcom.so`) | `/dev/media0`, `/dev/media1`, `/dev/video*`, `/dev/v4l-subdev*`, `/dev/synx_device`, `/sys/devices/soc0/soc_id` | `cdsp.{mdt,b*}`, `venus.{mdt,b*}` |
| **Sensors** (`android.hardware.sensors-service.xiaomi-multihal`, `sscrpcd`) | `/dev/fastrpc-*` (`adsprpc-smd`), `/dev/uhid`, `/sys/kernel/boot_adsp/boot`, `/mnt/vendor/persist/sensors/*` | `adsp.{mdt,b*}` |
| **Wi-Fi** (`android.hardware.wifi-service`, `wpa_supplicant`, `hostapd`) | `nl80211` netlink (`wlan0`, `p2p0`), `/dev/wlan`, `/dev/urandom` | `wlanmdsp.mbn`, `bdwlan.bin`, `Data.msc`, `/vendor/etc/wifi/WCNSS_qcom_cfg.ini` |
| **Bluetooth** (`android.hardware.bluetooth@1.0-service-qti`) | `/dev/ttyHS0`, `/dev/btpower`, `/dev/smd2`, `/dev/smd3`, `/sys/class/rfkill/rfkill*/*` | `tpbtfw00.tlv`, `tpnv00.bin` (or mainline `qca/crbtfw*.tlv`, `crnv*.bin`) |
| **Modem / RIL / IPA** (`qcrilNrd`, `rmt_storage`, `tftp_server`, `ipacm`, `netmgrd`) | `/dev/ipa`, `/dev/wwan_ioctl`, `/dev/ipaNatTable`, `/dev/rmnet_ctrl`, `/dev/diag`, `/dev/block/bootdevice/by-name/{modemst1,modemst2,fsg,fsc}`, QRTR sockets | `modem.{mdt,b*}`, `ipa_fws.{mdt,elf,b*}`, `mcfg_sw/*` |
| **NFC** (`android.hardware.nfc-service.nxp`, `init.veux.nfc.rc`) | `/dev/pn553` (or `/dev/nq-nci`) | NXP NFC firmware (in `/vendor/firmware`) |
| **Consumer IR** (`android.hardware.ir-service.lineage`) | `/dev/lirc0` | None |
| **Fingerprint** (`android.hardware.biometrics.fingerprint-service.xiaomi`, `fingerprint.fpc.default.so`) | `/dev/uinput`, `/dev/goodix_fp`, `/dev/silead_fp`, `/dev/qseecom`, `/dev/smcinvoke` | `fpctzapp64.mdt` (or `goodixfp.mdt`, `sileadta.mdt`) |
| **Vibrator** (`vendor.qti.hardware.vibrator.service`) | `/dev/aw8624_haptic`, `/sys/class/leds/vibrator/*` | `aw8624_haptic.bin`, `aw8624_rtp*.bin`, `aw8624_osc_rtp_24K_5s.bin` |
| **TEE / DRM / Keymaster** (`qseecomd`, `keymaster@4.1`, `gatekeeper@1.0`, `widevine`) | `/dev/qseecom`, `/dev/smcinvoke`, `/dev/ion` | `widevine.mbn`, `soter64.mdt`, `keymaster` |

---

## 4. Porting Effort Ranking & Suggested Bring-Up Order

### Effort Ranking (Lowest to Highest)
1. **Trivial (DT-only on `sm6375.dtsi`):** UART console (if pads accessible) / `pstore` ramoops, PM6125/PMR735A/PMK8350 regulators, `gpio-keys`, `pon_pwrkey`/`pon_resin`, `sdhc_2` (microSD), `usb_1` (USB2 peripheral/gadget for RNDIS/ADB), `tsens` thermal, `simple-framebuffer` (`0x85200000`).
2. **Low:** `remoteproc_{adsp,cdsp,mss}` + `rmtfs`, `wifi` (`ath10k_snoc` WCN3990), `bq27z561` fuel gauge, `bq25890` main charger, `rt1711h` TCPC, `bluetooth` (`hci_qca` on `uart1`), `gpu@5900000` (Adreno 619 + `gpucc`).
3. **Medium:** `ufshc` + `ufsphy` (`phy-qcom-qmp-ufs.c` SM6375/SC7280 table), `dispcc` + `mdss` + `dpu` + `dsi` DT wiring + `k6s_38_0c_0a` OLED DSC panel driver, `goodix_gtx8` touchscreen port, `aw8624` haptics port, `nxp-nci` NFC, `ir-spi` LIRC.
4. **High:** Audio (`bolero-cdc` + `wcd937x` + `aw882xx` smart PA + ADSP AudioReach/APR topology), `sc8551`/`ln8000` dual charge-pump fast charging, `venus` video decode/encode, `ipa` offload.
5. **Very High / Out-of-scope initially:** Camera (`camcc-sm6375` + TFE/OPE `camss`), proprietary TEE fingerprint (`fpc1020` requires Android/QSEE TZ).

### Suggested Bring-Up Order
1. **Phase 1 — Boot & Serial/USB Network:** Minimal `sm6375-xiaomi-veux.dts` with `simple-framebuffer`, `ramoops`, PMIC regulators, `sdhc_2`, `usb_1` (RNDIS/SSH), and `gpio-keys`. Boot via `fastboot boot` with `boot.img` / `vendor_boot`.
2. **Phase 2 — Storage & Connectivity:** Add UFS PHY/HC (`4804000.ufshc`), `remoteproc_*` + `rmtfs` (for modem/Wi-Fi/ADSP), `wifi` (`ath10k_snoc`), and `bluetooth` (`hci_qca`).
3. **Phase 3 — Display, GPU & Input:** Wire `dispcc` + `mdss` + `dpu` + `dsi` in `sm6375.dtsi`, port `k6s_38_0c_0a` panel driver (1080x2400@120Hz DSC), enable Adreno 619 (`gpu@5900000`), and port `goodix_gtx8` (`gt9896`) touchscreen.
4. **Phase 4 — Power & Peripherals:** Enable `bq27z561` + `bq25890` + `rt1711h`, then port `aw8624` haptics, `ir-spi`, NFC, and `sc8551` fast charging.
5. **Phase 5 — Audio & Multimedia:** Audio (`bolero` + `wcd937x` + `aw882xx`), `venus`, and `camss` (long-term).

---

## 5. Minimal First-Boot `sm6375-xiaomi-veux.dts` Skeleton

Based on mainline `arch/arm64/boot/dts/qcom/sm6375.dtsi` and `sm6375-sony-xperia-murray-pdx225.dts`, using exact memory regions (`veux-memory.dtsi`, `iomem.txt`), regulator bounds (`pm6125`/`pmr735a`), and bus/GPIO mappings from `veux.dts`:

```dts
// SPDX-License-Identifier: BSD-3-Clause
/dts-v1/;

/* PMK8350 is configured to use SID6 instead of 0 */
#define PMK8350_SID 6

#include <dt-bindings/gpio/gpio.h>
#include <dt-bindings/input/linux-event-codes.h>
#include "sm6375.dtsi"
#include "pm6125.dtsi"
#include "pmk8350.dtsi"
#include "pmr735a.dtsi"

/* PM6125 PON is used and we can't have duplicate labels */
/delete-node/ &pmk8350_pon;

/ {
	model = "Xiaomi Poco X4 Pro 5G";
	compatible = "xiaomi,veux", "qcom,sm6375";
	chassis-type = "handset";

	chosen {
		#address-cells = <2>;
		#size-cells = <2>;
		ranges;

		framebuffer: framebuffer@85200000 {
			compatible = "simple-framebuffer";
			reg = <0x0 0x85200000 0x0 0xc00000>;
			width = <1080>;
			height = <2400>;
			stride = <(1080 * 4)>;
			format = "a8r8g8b8";
			clocks = <&gcc GCC_DISP_AHB_CLK>,
				 <&gcc GCC_DISP_HF_AXI_CLK>,
				 <&gcc GCC_DISP_THROTTLE_CORE_CLK>,
				 <&gcc GCC_DISP_GPLL0_DIV_CLK_SRC>;
		};
	};

	gpio-keys {
		compatible = "gpio-keys";
		label = "gpio-keys";

		pinctrl-0 = <&vol_down_n>;
		pinctrl-names = "default";

		key-volume-down {
			label = "Volume Down";
			linux,code = <KEY_VOLUMEDOWN>;
			gpios = <&pmr735a_gpios 1 GPIO_ACTIVE_LOW>;
			debounce-interval = <15>;
			linux,can-disable;
			wakeup-source;
		};
	};

	reserved-memory {
		cont_splash_mem: memory@85200000 {
			reg = <0x0 0x85200000 0x0 0xc00000>;
			no-map;
		};

		/* Adjust size/addr to match bootloader ramoops if needed */
		ramoops@a3cca000 {
			compatible = "ramoops";
			reg = <0x0 0xa3cca000 0x0 0x60000>;
			record-size = <0x10000>;
			console-size = <0x30000>;
			pmsg-size = <0x20000>;
		};
	};
};

&pil_adsp_mem {
	reg = <0x0 0x86700000 0x0 0x2c00000>;
};

&pil_cdsp_mem {
	reg = <0x0 0x89300000 0x0 0x1e00000>;
};

&pil_video_mem {
	reg = <0x0 0x8b100000 0x0 0x500000>;
};

&pil_ipa_fw_mem {
	reg = <0x0 0x8b600000 0x0 0x10000>;
};

&pil_ipa_gsi_mem {
	reg = <0x0 0x8b610000 0x0 0xa000>;
};

&pil_gpu_micro_code_mem {
	reg = <0x0 0x8b61a000 0x0 0x2000>;
};

&gpi_dma0 {
	status = "okay";
};

&gpi_dma1 {
	status = "okay";
};

&pmk8350_adc_tm {
	status = "okay";
};

&pmk8350_rtc {
	status = "okay";
};

&pmr735a_gpios {
	vol_down_n: vol-down-n-state {
		pins = "gpio1";
		function = "normal";
		power-source = <1>;
		bias-pull-up;
		input-enable;
	};
};

&pon_pwrkey {
	status = "okay";
};

&pon_resin {
	linux,code = <KEY_VOLUMEUP>;
	status = "okay";
};

&qupv3_id_0 {
	status = "okay";
};

&qupv3_id_1 {
	status = "okay";
};

&i2c0 {
	clock-frequency = <400000>;
	status = "okay";

	fuel-gauge@55 {
		compatible = "ti,bq27z561";
		reg = <0x55>;
	};
};

&i2c10 {
	clock-frequency = <400000>;
	status = "okay";
	/* bq25890@6a, rt1711h@4e, sn-nci@28, aw882xx@34/35 */
};

&remoteproc_adsp {
	firmware-name = "qcom/sm6375/xiaomi/veux/adsp.mbn";
	status = "okay";
};

&remoteproc_cdsp {
	firmware-name = "qcom/sm6375/xiaomi/veux/cdsp.mbn";
	status = "okay";
};

&remoteproc_mss {
	firmware-name = "qcom/sm6375/xiaomi/veux/modem.mbn";
	status = "okay";
};

&rpm_requests {
	regulators-0 {
		compatible = "qcom,rpm-pm6125-regulators";

		pm6125_s5: s5 {
			regulator-min-microvolt = <382000>;
			regulator-max-microvolt = <1120000>;
		};
		pm6125_s6: s6 {
			regulator-min-microvolt = <320000>;
			regulator-max-microvolt = <1374000>;
		};
		pm6125_s7: s7 {
			regulator-min-microvolt = <1574000>;
			regulator-max-microvolt = <2040000>;
		};
		pm6125_l2: l2 {
			regulator-min-microvolt = <1170000>;
			regulator-max-microvolt = <1304000>;
		};
		pm6125_l5: l5 {
			regulator-min-microvolt = <1650000>;
			regulator-max-microvolt = <2960000>;
			regulator-allow-set-load;
		};
		pm6125_l7: l7 {
			regulator-min-microvolt = <880000>;
			regulator-max-microvolt = <880000>;
		};
		pm6125_l9: l9 {
			regulator-min-microvolt = <1504000>;
			regulator-max-microvolt = <2000000>;
		};
		pm6125_l10: l10 {
			regulator-min-microvolt = <1620000>;
			regulator-max-microvolt = <1980000>;
		};
		pm6125_l13: l13 {
			regulator-min-microvolt = <1650000>;
			regulator-max-microvolt = <1980000>;
		};
		pm6125_l16: l16 {
			regulator-min-microvolt = <1620000>;
			regulator-max-microvolt = <1980000>;
		};
		pm6125_l21: l21 {
			regulator-min-microvolt = <3000000>;
			regulator-max-microvolt = <3312000>;
		};
		pm6125_l22: l22 {
			regulator-min-microvolt = <2704000>;
			regulator-max-microvolt = <2960000>;
			regulator-allow-set-load;
		};
		pm6125_l23: l23 {
			regulator-min-microvolt = <3000000>;
			regulator-max-microvolt = <3312000>;
		};
		pm6125_l24: l24 {
			regulator-min-microvolt = <2704000>;
			regulator-max-microvolt = <3544000>;
		};
	};

	regulators-1 {
		compatible = "qcom,rpm-pmr735a-regulators";

		pmr735a_l2: l2 {
			regulator-min-microvolt = <640000>;
			regulator-max-microvolt = <640000>;
		};
		pmr735a_l7: l7 {
			regulator-min-microvolt = <2700000>;
			regulator-max-microvolt = <3544000>;
		};
	};
};

&sdhc_2 {
	status = "okay";
	vmmc-supply = <&pm6125_l22>;
	vqmmc-supply = <&pm6125_l5>;
	cd-gpios = <&tlmm 94 GPIO_ACTIVE_HIGH>;
};

&usb_1 {
	status = "okay";
};

&usb_1_dwc3 {
	dr_mode = "peripheral";
};

&usb_1_hsphy {
	vdda-pll-supply = <&pm6125_l7>;
	vdda18-supply = <&pm6125_l10>;
	vdda33-supply = <&pmr735a_l7>;
	status = "okay";
};

&wifi {
	vdd-0.8-cx-mx-supply = <&pmr735a_l2>;
	vdd-1.8-xo-supply = <&pm6125_l16>;
	vdd-1.3-rfa-supply = <&pm6125_l2>;
	vdd-3.3-ch0-supply = <&pm6125_l23>;
	vdd-3.3-ch1-supply = <&pm6125_l21>;
	status = "okay";
};

&xo_board_clk {
	clock-frequency = <19200000>;
};
```
