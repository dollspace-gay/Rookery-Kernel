---
title: "Tier-2: drivers/media — Media subsystem (V4L2 + media-controller + DVB + CEC + RC + camera ISP + USB-video + PCI-capture)"
tags: ["tier-2", "drivers", "media", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Tier-2 overview for the Linux Media subsystem — V4L2 (Video for Linux 2) + media-controller + DVB (Digital Video Broadcasting) + CEC (Consumer Electronics Control) + RC (Remote Control / IR) + camera ISP (Image Signal Processor) + per-vendor capture/playback drivers (USB webcams, USB-DTV tuners, PCI-capture cards, hardware H.264/HEVC encoders/decoders, USB-Audio MIDI capture, FireWire video, FM/AM radio receivers).

Six concentric architectural rings:

1. **V4L2 core**: `/dev/video<N>` + `/dev/v4l-subdev<N>` + `/dev/swradio<N>` + `/dev/vbi<N>` + `/dev/v4l-touch<N>` chardev IOCTLs, video-buffer allocation framework (videobuf2 — `vb2_*`), v4l2-mem2mem framework for stateful codec drivers, ctrl framework (per-driver control hierarchy: brightness, contrast, exposure, white-balance, codec-bitrate, etc.), event framework, fh framework (per-fd state), tuner core (analog tuner abstraction), DV-timings (HDMI/SDI digital-video timings), VBI (Vertical Blanking Interval — captioning + teletext)
2. **Media controller**: `/dev/media<N>` device representing the topology of a media pipeline (entities + pads + links); used by complex camera ISPs to expose configurable pipeline graphs to userspace
3. **DVB core**: `/dev/dvb/adapter<N>/{frontend<N>, demux<N>, dvr<N>, ca<N>, net<N>}` — Digital TV stack with frontend demodulators, hardware MPEG-TS demuxers, DVR for raw stream recording, conditional-access for encrypted broadcasts, and DVB networking (MPE encapsulation)
4. **CEC core**: `/dev/cec<N>` — HDMI Consumer Electronics Control (one-touch-play, system-standby, audio-routing, remote-control over HDMI). Tightly coupled to DRM HDMI connectors
5. **Remote Control core**: `/dev/lirc<N>` + IR-keymap input-event injection. Keymaps for ~200+ IR remotes
6. **Per-driver subsystems**: USB camera drivers (uvcvideo, gspca, pwc, em28xx, go7007, hdpvr), PCI capture cards (cx18, cx88, ivtv, saa7164, bt8xx, ngene, dt3155, mantis, dvb-* PCI tuners), FireWire camera (firedtv), I2C tuner ICs, SPI tuner ICs, MMC/SDIO video, ARM SoC ISPs (allegro, atomisp, cadence, chips-media, exynos4-is, intel/ipu3 + ipu6, mediatek/vcodec + jpeg + camsv, qcom/camss + venus + iris, rockchip/rga + rkisp1 + rkvdec, sti/{bdisp,c8sectpfe,delta,hva}, sunxi/cedrus, ti/davinci + cal + j721e-csi2rx + omap3isp + omap-vpe + ti-csi2rx + ti-vpe, vimc test driver)

Heavily cross-referenced from `drivers/usb/00-overview.md` (USB webcams + USB DTV tuners), `drivers/pci/00-overview.md` (PCI capture cards), `drivers/i2c/`, `drivers/spi/` (sensor/tuner ICs), `kernel/dma/00-overview.md` (videobuf2 dma-buf + dma-contig), `drivers/gpu/drm/00-overview.md` (dma-buf cross-driver share — V4L2 → DRM zero-copy), `sound/00-overview.md` (HDA-codec on capture pipelines), `drivers/hid/00-overview.md` (RC injects input events).

### Out of Scope

- Per-Tier-3 (Phase D) — per-driver docs arrive incrementally
- ARM-only SoC ISP/codec drivers (`platform/allegro-dvt`, `platform/amphion`, `platform/atmel/`, `platform/cadence/`, `platform/chips-media/`, `platform/exynos4-is`, `platform/marvell/`, `platform/mediatek/`, `platform/microchip/`, `platform/nuvoton/`, `platform/nvidia/`, `platform/nxp/`, `platform/qcom/`, `platform/renesas/`, `platform/rockchip/`, `platform/samsung/`, `platform/sti/`, `platform/sunxi/`, `platform/synopsys/`, `platform/ti/`, `platform/verisilicon/`, `platform/via/`, `platform/vimc`) — keep present but compile-gated off for v0 (most have no x86_64 use case)
- Most legacy gspca / em28xx vendor variants (keep representative subset)
- 32-bit-only paths
- Implementation code

### components

### Core

- **v4l2-core** (`v4l2-core/`): V4L2 base — `v4l2-dev.c`, `v4l2-fh.c`, `v4l2-ioctl.c`, `v4l2-event.c`, `v4l2-ctrls-*`, `v4l2-mem2mem.c`, `v4l2-subdev.c`, `videobuf2-{core,vmalloc,dma-contig,dma-sg,memops,v4l2}.c`, `tuner-core.c`, `v4l2-async.c`, `v4l2-fwnode.c`, `v4l2-flash-led-class.c`, `v4l2-i2c.c`, `v4l2-spi.c`, `v4l2-jpeg.c`, `v4l2-vp9.c`, `v4l2-h264.c`, `v4l2-hevc.c`, `v4l2-av1.c` (stateless codec helpers), `v4l2-mc.c` (media-controller integration), `v4l2-dv-timings.c` (DV-timings), `v4l2-rect.c` (rectangle math), `v4l2-trace.c`
- **mc** (`mc/`): media-controller core — `mc-device.c`, `mc-entity.c`, `mc-link.c`, `mc-request.c` (request API for atomic camera config)
- **dvb-core** (`dvb-core/`): DVB base — `dvbdev.c`, `dvb_demux.c`, `dvb_frontend.c`, `dmxdev.c`, `dvb_ca_en50221.c`, `dvb_net.c`, `dvb_ringbuffer.c`, `dvb_math.c`, `dvb_vb2.c`
- **rc** (`rc/`): remote-control core + per-protocol decoders + per-keymap tables + per-driver IR receivers (cx23885-input, ene_ir, igorplugusb, imon, mceusb, redrat3, streamzap, winbond-cir, etc.)
- **cec** (`cec/`): CEC framework + cec-pin (low-level GPIO bit-banging) + per-controller drivers
- **common** (`common/`): shared between v4l2 + dvb (saa7146 PCI helpers, cx2341x MPEG-encoder helpers, b2c2-flexcop helpers, cypress firmware helpers, siano shared, tveeprom, videobuf-* legacy)
- **firewire** (`firewire/`): IIDC + DV1394 + AVC capture (legacy)

### Hardware sub-trees

- **i2c** (`i2c/`): I2C-attached image sensors (~150 chips: ov2680, ov5640, ov7670, ov13858, imx214, imx274, imx290, imx412, imx477, imx519, ar0521, ar0830, hi556, hi846, gc0308, gc2145, gc8034, mt9m001, mt9m032, mt9m111, mt9p031, mt9t001, mt9t112, mt9v011, mt9v032, mt9v111, max9286, ov2659, ov4689, ov6650, ov7251, ov7740, ov772x, ov8856, ov8858, ov8865, ov9282, ov9640, ov9650, ov9734, ov9740, rj54n1cb0c, s5c73m3, s5k4ecgx, s5k5baf, s5k6a3, s5k6aa, sr030pc30, ths7303, ths8200, tw5864, tw686x, tw9903, tw9906, tw9910), I2C-attached video encoders/decoders (saa7115, saa7127, tvp514x, tvp7002), I2C-attached HDMI receivers (adv7180, adv7183, adv7393, adv7393, adv748x, adv7511, adv7604, adv7842, ad9389b, tc358743, tc358840, tda1997x, tda9840), audio I2C codecs for capture (cs5345, cs53l32a, msp3400, tlv320aic23b, vp27smpx, wm8739, wm8775, wm8995, wm8996, m52790)
- **pci** (`pci/`): PCI capture/decode cards — bt8xx (legacy capture), cx18, cx23885, cx25821, cx88, ddbridge, dm1105, dt3155, ivtv (Hauppauge PVR), mantis, ngene, pluto2, saa7134, saa7164, smipcie, sta2x11, tw5864, tw686x, tw68, intel/ipu3-cio2 + ipu3-imgu, intel/ipu6 (newer Intel image-processing-unit)
- **platform** (`platform/`): per-SoC ISP/codec drivers (allegro-dvt, amphion, atmel/atmel-isc + atmel-isi, cadence/cdns-csi2{rx,tx}, chips-media/wave5 + coda, exynos4-is, intel/ipu6, marvell/mcam-core + mmp-driver, mediatek/vcodec + jpeg + mdp + mdp3 + camsv + isp_50 + iris, microchip/microchip-csi2dc + xisc, nuvoton/npcm-video, nvidia/tegra-vde, nxp/dw100 + imx-jpeg + imx-mipi-csis + imx7-media-csi + imx-pxp, qcom/camss + venus + iris + ipa, renesas/rcar-csi2 + rcar-isp + rcar-vin + rcar_drif + rcar_fcp + rcar_jpu + rzg2l-{cru,csi2,vin}, rockchip/rga + rkisp1 + rkvdec, samsung/{exynos-gsc, exynos4-is, mfc, s3c-camif, s5p-cec, s5p-jpeg, s5p-mfc, s5p-tv}, sti/{bdisp, c8sectpfe, delta, hva}, sunxi/cedrus, synopsys/dw-mipi-csi2 + dwc-hdmi-cec, ti/davinci + cal + j721e-csi2rx + omap3isp + omap_vout + ti-csi2rx + ti-vpe, verisilicon, via/marvell-ccic, vimc)
- **radio** (`radio/`): FM/AM radio receivers — radio-* (cadet, gemtek, isa, maxiradio, miropcm20, rtrack, sf16fmi, sf16fmr2, terratec, trust, typhoon, zoltrix), si4713 (FM-TX), si470x (USB FM), si4711 (USB FM), tea5764, mr800, dsbr100, keene, ma901, raremono, shark, shark2, si4713, tea575x
- **rc** (`rc/`): remote-control IR receivers — ati_remote, ene_ir, fintek-cir, gpio-ir-recv + gpio-ir-tx, hix5hd2_ir, igorplugusb, iguanair, img-ir, imon, ir-hix5hd2, ir-imon-decoder, ir-jvc-decoder, ir-mce_kbd-decoder, ir-nec-decoder, ir-rc5-decoder, ir-rc6-decoder, ir-rcmm-decoder, ir-sanyo-decoder, ir-sharp-decoder, ir-sony-decoder, ir-spi, ir-xmp-decoder, lirc_dev, mceusb, meson-ir + meson-ir-tx, mtk-cir, nuvoton-cir, pwm-ir-tx, redrat3, serial_ir, sir_ir, st_rc, streamzap, sunxi-cir, ttusbir, winbond-cir, xbox_remote
- **usb** (`usb/`): USB-attached video — uvcvideo (UVC class — bulk of webcams), em28xx (Empia capture), gspca/* (legacy mass-vendor webcams), pvrusb2, dvb-usb-v2, hdpvr, pwc, s2255drv, stk1160, tm6000, usbtv, zr364xx, etc.
- **mmc** (`mmc/`): SDIO-attached video (legacy)
- **firewire** (`firewire/`): IEEE-1394 video (legacy)
- **spi** (`spi/`): SPI-attached video sensors (gs1662)
- **tuners** (`tuners/`): RF tuner ICs (~30+) — fc0011, fc0012, fc0013, fc2580, m88rs6000t, max2165, mc44s803, mt2060/63/131/266, mt20xx, mxl301rf, mxl5005s, mxl5007t, mxl301rf, qm1d1c0042, qm1d1b0004, qt1010, r820t, si2157, tda18212, tda18218, tda18250, tda18271, tda827x, tda8290, tda9887, tea5761, tea5767, tua9001, tuner-i2c, tuner-types, tuner-xc2028, tuner-simple, xc4000, xc5000
- **dvb-frontends** (`dvb-frontends/`): DVB demodulator chips (~150+) — a8293, af9013, af9033, as102, ascot2e, atbm8830, au8522, av7110, bcm3510, cx22700, cx22702, cx24110, cx24116, cx24117, cx24120, cx24123, cx2 4140, cxd2099, cxd2820r, cxd2841er, cxd2880, dib0070, dib0090, dib3000, dib7000, dib8000, dib9000, dibx000_common, drx39xyj, drxd, drxk, ds3000, dst, dtt200u, dvb-pll, dvb_dummy_fe, ec100, gp8psk-fe, helene, horus3a, isl6405, isl6421, isl6423, ix2505v, l64781, lg2160, lgdt3305, lgdt3306a, lgdt330x, lgs8gxx, lnbh24, lnbh25, lnbh29, lnbp21, lnbp22, m88ds3103, m88rs2000, mb86a16, mb86a20s, mn88472, mn88473, mt312, mt352, mxl5xx, mxl692, nxt200x, nxt6000, or51132, or51211, plxsdsl, push2tv, rtl2830, rtl2832, rtl2832_sdr, s5h1409, s5h1411, s5h1432, s921, si2165, si2168, si21xx, sp2, sp887x, stb0899, stb6000, stb6100, stv0297, stv0299, stv0367, stv0900, stv0910, stv090x, stv6110, stv6110x, stv6111, tc90522, tda10021, tda10023, tda10048, tda1004x, tda10071, tda10086, tda18271c2dd, tda665x, tda8083, tda8261, tda826x, tdhd1, ts2020, tua6100, ves1820, ves1x93, zd1301_demod, zl10036, zl10039, zl10353

### scope

This Tier-2 governs `/home/doll/linux-src/drivers/media/` (~30 subdirs covering thousands of files), public headers `include/media/` (~150 headers), UAPI `include/uapi/linux/videodev2.h` + `include/uapi/linux/v4l2-*.h` + `include/uapi/linux/dvb/*.h` + `include/uapi/linux/cec*.h` + `include/uapi/linux/lirc.h` + `include/uapi/linux/dma-heap.h` (cross-ref `kernel/dma/00-overview.md`).

### compatibility contract — outline

### `/dev/video<N>`, `/dev/v4l-subdev<N>`, `/dev/vbi<N>`, `/dev/swradio<N>`, `/dev/v4l-touch<N>` chardev IOCTLs

V4L2 IOCTLs byte-identical: `VIDIOC_QUERYCAP`, `_ENUM_FMT`, `_G_FMT`, `_S_FMT`, `_TRY_FMT`, `_REQBUFS`, `_QUERYBUF`, `_QBUF`, `_DQBUF`, `_PREPARE_BUF`, `_EXPBUF`, `_CREATE_BUFS`, `_QUERYCTRL`, `_QUERY_EXT_CTRL`, `_QUERYMENU`, `_G_CTRL`, `_S_CTRL`, `_G_EXT_CTRLS`, `_S_EXT_CTRLS`, `_TRY_EXT_CTRLS`, `_G_TUNER`, `_S_TUNER`, `_G_AUDIO`, `_S_AUDIO`, `_G_INPUT`, `_S_INPUT`, `_ENUMINPUT`, `_G_OUTPUT`, `_S_OUTPUT`, `_ENUMOUTPUT`, `_G_AUDOUT`, `_S_AUDOUT`, `_G_MODULATOR`, `_S_MODULATOR`, `_G_FREQUENCY`, `_S_FREQUENCY`, `_CROPCAP`, `_G_CROP`, `_S_CROP`, `_G_SELECTION`, `_S_SELECTION`, `_G_JPEGCOMP`, `_S_JPEGCOMP`, `_G_PARM`, `_S_PARM`, `_G_STD`, `_S_STD`, `_ENUMSTD`, `_QUERYSTD`, `_ENUM_FRAMESIZES`, `_ENUM_FRAMEINTERVALS`, `_G_PRIORITY`, `_S_PRIORITY`, `_LOG_STATUS`, `_G_DV_TIMINGS`, `_S_DV_TIMINGS`, `_QUERY_DV_TIMINGS`, `_ENUM_DV_TIMINGS`, `_DV_TIMINGS_CAP`, `_SUBSCRIBE_EVENT`, `_UNSUBSCRIBE_EVENT`, `_DQEVENT`, `_STREAMON`, `_STREAMOFF`, `_G_EDID`, `_S_EDID`, `_G_ENC_INDEX`, `_ENCODER_CMD`, `_TRY_ENCODER_CMD`, `_DECODER_CMD`, `_TRY_DECODER_CMD`, `_QUERY_DV_PRESET`, `_DBG_S_REGISTER`, `_DBG_G_REGISTER`, `_DBG_G_CHIP_INFO`, `_OVERLAY`, `_G_FBUF`, `_S_FBUF`, `_REQBUFS`, etc.

Wire format byte-identical so v4l2-utils, GStreamer v4l2src/v4l2sink, ffmpeg v4l2 device, libcamera, OBS Studio v4l2 capture, Cheese, Zoom/Skype consume unchanged.

### `/dev/media<N>` IOCTLs

Media controller: `MEDIA_IOC_DEVICE_INFO`, `_ENUM_ENTITIES`, `_ENUM_LINKS`, `_ENUM_PADS`, `_SETUP_LINK`, `_G_TOPOLOGY`, `_REQUEST_ALLOC`, `_REQUEST_QUEUE`, `_REQUEST_REINIT`. Wire format byte-identical so libcamera + libmediactl consume unchanged.

### `/dev/dvb/adapter<N>/{frontend<N>, demux<N>, dvr<N>, ca<N>, net<N>}` IOCTLs

DVB IOCTLs byte-identical: `FE_GET_INFO`, `FE_READ_STATUS`, `FE_GET_PROPERTY`, `FE_SET_PROPERTY`, `FE_DISEQC_*`, `FE_SET_FRONTEND_TUNE_MODE`, `FE_GET_EVENT`, `FE_DISEQC_RESET_OVERLOAD`, `FE_DISEQC_SEND_MASTER_CMD`, `FE_DISEQC_RECV_SLAVE_REPLY`, `DMX_*` (set/start/stop filter, etc.), `CA_*`. Wire format byte-identical so dvb-tools (dvbv5-zap, dvbv5-scan, w_scan), TVHeadend, MythTV, kaffeine consume unchanged.

### `/dev/cec<N>` IOCTLs

CEC IOCTLs byte-identical: `CEC_ADAP_G_CAPS`, `CEC_ADAP_S_LOG_ADDRS`, `CEC_ADAP_G_LOG_ADDRS`, `CEC_ADAP_S_PHYS_ADDR`, `CEC_ADAP_G_PHYS_ADDR`, `CEC_ADAP_G_CONNECTOR_INFO`, `CEC_TRANSMIT`, `CEC_RECEIVE`, `CEC_DQEVENT`, `CEC_G_MODE`, `CEC_S_MODE`. Wire format byte-identical so cec-ctl + cec-tools consume.

### `/dev/lirc<N>` IOCTLs

LIRC IOCTLs (RC framework chardev): `LIRC_GET_FEATURES`, `_GET_REC_MODE`, `_SET_REC_MODE`, `_GET_REC_RESOLUTION`, `_GET_MIN_TIMEOUT`, `_GET_MAX_TIMEOUT`, `_SET_REC_TIMEOUT`, `_GET_REC_TIMEOUT`, `_SET_REC_TIMEOUT_REPORTS`, `_GET_REC_CARRIER`, `_SET_REC_CARRIER`, `_SET_SEND_CARRIER`, `_SET_SEND_DUTY_CYCLE`, `_SET_TRANSMITTER_MASK`, `_SET_REC_CARRIER_RANGE`, `_SET_WIDEBAND_RECEIVER`. Wire format byte-identical so lircd consumes.

### sysfs surface

Per-device entries under `/sys/class/video4linux/video<N>/`, `/sys/class/dvb/dvb<N>.<entity>/`, `/sys/class/cec/cec<N>/`, `/sys/class/rc/rc<N>/`, `/sys/class/swradio/swradio<N>/`. Layout + content byte-identical so v4l2-info / dvbnet-info consume.

### dma-buf integration

V4L2 buffers can be exported as dma-buf (`VIDIOC_EXPBUF`) and imported (`V4L2_MEMORY_DMABUF`). Cross-ref `kernel/dma/00-overview.md` for buffer-share semantics.

### Codec mem2mem framework

Stateful + stateless codecs (H.264, HEVC, VP8, VP9, AV1, JPEG, FWHT) use `v4l2_m2m_*` framework with output-queue (encoded → decoded for decoders) and capture-queue. UAPI byte-identical for ffmpeg / GStreamer / libavcodec.

### Tracepoints

`events/v4l2/`, `events/cec/`, `events/dvb/` byte-identical names + format.

### tier-3 docs governed by this tier-2

(Phase D will add these incrementally.)

| Tier-3 doc | Scope |
|---|---|
| `drivers/media/v4l2-core.md` | `v4l2-core/`: V4L2 base — chardev + IOCTL dispatch + fh + ctrls + event |
| `drivers/media/videobuf2.md` | `v4l2-core/videobuf2-*`: videobuf2 buffer-mgmt + memops |
| `drivers/media/v4l2-mem2mem.md` | `v4l2-core/v4l2-mem2mem.c` + per-codec helpers (h264/hevc/av1/vp9/jpeg) |
| `drivers/media/v4l2-subdev.md` | `v4l2-core/v4l2-subdev.c`: subdev framework + async + fwnode |
| `drivers/media/v4l2-ctrls.md` | `v4l2-core/v4l2-ctrls-*`: ctrl framework |
| `drivers/media/v4l2-flash-led.md` | `v4l2-core/v4l2-flash-led-class.c`: flash-LED integration |
| `drivers/media/v4l2-mc-integration.md` | `v4l2-core/v4l2-mc.c`: media-controller ↔ V4L2 bridge |
| `drivers/media/mc-core.md` | `mc/`: media-controller — entities + pads + links + request API |
| `drivers/media/dvb-core.md` | `dvb-core/`: DVB chardev + frontend + demux + ca + net |
| `drivers/media/cec-core.md` | `cec/`: CEC framework + cec-pin |
| `drivers/media/rc-core.md` | `rc/`: RC framework + per-protocol decoders + per-keymap tables |
| `drivers/media/usb-uvc.md` | `usb/uvc/`: UVC class (the dominant USB webcam path) |
| `drivers/media/usb-em28xx.md` | `usb/em28xx/`: Empia EM28xx USB capture |
| `drivers/media/usb-gspca.md` | `usb/gspca/`: legacy mass-vendor USB webcams |
| `drivers/media/usb-pwc.md` | `usb/pwc/`: Philips USB webcams (legacy) |
| `drivers/media/pci-cx18.md` | `pci/cx18/`: Conexant CX23418 capture |
| `drivers/media/pci-cx88.md` | `pci/cx88/`: Conexant CX88 capture |
| `drivers/media/pci-cx23885.md` | `pci/cx23885/`: Conexant CX23885 capture |
| `drivers/media/pci-ivtv.md` | `pci/ivtv/`: Hauppauge PVR cards |
| `drivers/media/pci-saa7164.md` | `pci/saa7164/`: NXP SAA7164 capture |
| `drivers/media/pci-bt8xx.md` | `pci/bt8xx/`: BT848/878 (legacy) |
| `drivers/media/pci-intel-ipu3.md` | `pci/intel/ipu3/`: Intel IPU3 (Skylake+) image-processing-unit |
| `drivers/media/pci-intel-ipu6.md` | `pci/intel/ipu6/`: Intel IPU6 (Tiger Lake+) IPU |
| `drivers/media/i2c-sensors.md` | `i2c/`: image sensors (~150 chips) |
| `drivers/media/i2c-encoders-decoders.md` | `i2c/`: video encoders/decoders + HDMI receivers |
| `drivers/media/dvb-frontends.md` | `dvb-frontends/`: DVB demodulator chips |
| `drivers/media/tuners.md` | `tuners/`: RF tuner ICs |
| `drivers/media/radio.md` | `radio/`: FM/AM radio receivers |
| `drivers/media/firewire.md` | `firewire/`: IEEE-1394 video (legacy) |

### compatibility outline (top-level)

- REQ-O1: All `/dev/video<N>` + `/dev/v4l-subdev<N>` + `/dev/vbi<N>` + `/dev/swradio<N>` + `/dev/v4l-touch<N>` IOCTLs byte-identical (v4l2-utils + libcamera + GStreamer + ffmpeg + OBS + Cheese consume unchanged).
- REQ-O2: `/dev/media<N>` media-controller IOCTLs byte-identical.
- REQ-O3: `/dev/dvb/adapter<N>/{frontend,demux,dvr,ca,net}<N>` IOCTLs byte-identical (dvb-tools + TVHeadend consume unchanged).
- REQ-O4: `/dev/cec<N>` IOCTLs byte-identical (cec-ctl consumes unchanged).
- REQ-O5: `/dev/lirc<N>` IOCTLs byte-identical (lircd consumes unchanged).
- REQ-O6: videobuf2 + v4l2-m2m UAPI byte-identical (codec drivers + GStreamer m2m elements work unchanged).
- REQ-O7: dma-buf import/export across V4L2 ↔ DRM ↔ DMA-heap byte-identical (zero-copy pipelines work).
- REQ-O8: Per-class sysfs surface byte-identical.
- REQ-O9: Per-driver `MODULE_DEVICE_TABLE` modaliases byte-identical for all in-tree consumed drivers.
- REQ-O10: Tracepoints under `events/v4l2/`, `events/cec/`, `events/dvb/` byte-identical names + format.
- REQ-O11: TLA+ models declared at this Tier-2 (videobuf2 queue state machine, v4l2-m2m source/dest queue ordering, media-controller request API atomic-apply, dvb-frontend tune state machine, CEC physical-address tree consistency).
- REQ-O12: Verus/Creusot Layer-4 functional contracts on V4L2 buffer refcount + plane-format size arithmetic + DVB section parser bounds.
- REQ-O13: Hardening: row-1 features applied per `00-security-principles.md`, with media-specific reinforcement (per-device CAP gating; dma-buf import LSM mediation for cross-process video share; cec PHYS_ADDR/LOG_ADDR sanitization).

### acceptance criteria (top-level)

- [ ] AC-O1: USB webcam (UVC) enumerates as `/dev/video<N>`; `v4l2-ctl --list-formats-ext` matches upstream baseline. (covers REQ-O1)
- [ ] AC-O2: Cheese / OBS / Zoom captures from UVC webcam. (covers REQ-O1, REQ-O6)
- [ ] AC-O3: GStreamer pipeline `v4l2src ! videoconvert ! kmssink` zero-copies camera → display via dma-buf. (covers REQ-O7)
- [ ] AC-O4: ffmpeg `-c:v h264_v4l2m2m` encode on a stateful V4L2 codec driver works. (covers REQ-O6)
- [ ] AC-O5: dvbv5-zap on DVB-S2 tuner achieves lock + receives MPEG-TS; `dvbv5-zap channel.conf` + tssniffer captures stream. (covers REQ-O3)
- [ ] AC-O6: cec-ctl test: HDMI-CEC TV detected, `cec-ctl -p 1.0.0.0 --tv` enumerates audio + remote functions. (covers REQ-O4)
- [ ] AC-O7: ir-keytable test: IR remote button-press emits correct keycode via `/dev/lirc0` + input-event device. (covers REQ-O5)
- [ ] AC-O8: libcamera test: pipeline-handler enumerates cameras via media-controller and captures stream. (covers REQ-O2, REQ-O6)
- [ ] AC-O9: Intel IPU3/6 test on Tiger Lake reference HW: front-camera enumerates, libcamera captures preview. (covers REQ-O1, REQ-O2)
- [ ] AC-O10: kselftest media tests pass. (covers REQ-O11, REQ-O12)
- [ ] AC-O11: v4l2-compliance + dvb-fe-tool + cec-compliance test suites pass on every supported in-tree v0 driver. (covers REQ-O11)
- [ ] AC-O12: dma-buf tex-from-video test: GBM imports V4L2 dma-buf as DRM framebuffer. (covers REQ-O7)

### verification (top-level)

### Layer 2: TLA+ models — mandatory list

| Model | Owned by |
|---|---|
| `models/media/vb2_queue.tla` | `drivers/media/videobuf2.md` (proves: videobuf2 queue state machine — REQBUFS → QBUF → STREAMON → DQBUF → STREAMOFF; concurrent qbuf + dqbuf + streamon/off never produce stuck buffer or skipped buffer; per-buf state transitions monotonic) |
| `models/media/v4l2_m2m.tla` | `drivers/media/v4l2-mem2mem.md` (proves: mem2mem source-queue and capture-queue interaction — encoder/decoder driver dispatches when both queues have buffers; concurrent enqueue from both sides + driver job-completion never produce starved queue or duplicated job) |
| `models/media/mc_request.tla` | `drivers/media/mc-core.md` (proves: media-controller request API atomic-apply — userspace builds a request with N controls + queues, request is applied atomically at next frame boundary; failed apply rolls back with no partial state) |
| `models/media/dvb_fe_tune.tla` | `drivers/media/dvb-core.md` (proves: DVB-frontend tune state machine — IDLE → TUNING → LOCKED → STREAMING; concurrent FE_SET_PROPERTY tune + FE_GET_EVENT readers properly synchronize; lock/unlock signal observed exactly once per state transition) |
| `models/media/cec_phys_addr.tla` | `drivers/media/cec-core.md` (proves: HDMI CEC physical-address tree consistency — when sink HDMI-DRM connector reports new EDID PA, all CEC adapters under that connector reconfigure logical-address claim atomically; no cross-tree address collision) |

### Layer 4: Functional contracts (Verus / Creusot)

| Component | Contract topic |
|---|---|
| `drivers/media/v4l2-core.md` | `v4l2_ioctl` dispatch invariant: per-driver `v4l2_ioctl_ops` indexed by ioctl-cmd; unsupported ioctl returns -ENOTTY; per-fd state validated against device-state |
| `drivers/media/videobuf2.md` | vb2 buffer refcount invariant: each buf has refcount that goes 0 only at QBUF-cancel or queue-release; mmap'd buffers hold extra refcount until munmap |
| `drivers/media/dvb-core.md` | DVB section-filter parser bounds: every section length validated against MPEG-TS spec max (4096); CRC32 verified before delivery to userspace; truncated section dropped |

### hardening (top-level)

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this Tier-2

| Feature | Default inherited from |
|---|---|
| **REFCOUNT** | per-video_device + per-vb2_buffer + per-v4l2_fh + per-dvb_demux + per-cec_adapter + per-rc_dev refcounts use `Refcount` (saturating) | § Mandatory |
| **CONSTIFY** | per-driver `v4l2_ioctl_ops`, `vb2_ops`, `cec_adap_ops`, `dvb_frontend_ops`, `rc_ir_raw_ops` `static const` | § Mandatory |
| **SIZE_OVERFLOW** | per-frame size = width×height×bpp, plane-pitch×plane-height arithmetic + DVB section-length + per-buf user-pointer length use checked operators | § Mandatory |
| **RAP / FORTIFY_RW** | per-driver v4l2_ioctl_ops + per-driver vb2_ops + per-driver dvb_frontend_ops vtables read-only-after-init | § Mandatory |
| **MEMORY_SANITIZE** | freed video frames + DVB sections + CEC messages cleared (carries camera frames + decrypted media content) | § Default-on configurable off |
| **STRICT_KERNEL_RWX** | media has no JIT — N/A | § Mandatory |

### Row-2 (LSM-stackable) features for media

LSM hooks called: file-LSM hooks on `/dev/video*`, `/dev/v4l-subdev*`, `/dev/media*`, `/dev/dvb/*`, `/dev/cec*`, `/dev/lirc*`, `/dev/swradio*` open + ioctl. `security_v4l2_*` family (proposed for Rookery — upstream coverage minimal).

GR-RBAC adds:
- Per-role disallow `/dev/video*` open (denies camera access — defends against background spying).
- Per-role disallow `/dev/lirc*` open (denies arbitrary IR transmit — could control nearby devices).
- Per-role disallow `/dev/dvb/*` open (denies DVB tuner access).
- Per-role disallow `/dev/swradio*` open (denies SDR receiver access — defends against unauthorized RF capture).
- Per-role audit of every camera `STREAMON` (which uid started capture, on which device).

### Media-specific reinforcement

- **Camera "capture-active" indicator**: when V4L2 device is in STREAMING state, kernel optionally drives a per-platform capture-LED (already true on some platforms via v4l2-flash-led-class; Rookery makes it consistent).
- **Per-process camera time quota**: optional CONFIG_V4L2_CAMERA_TIMEQUOTA — per-userns daily-cumulative camera-streaming-time cap (defense against background process keeping camera stream open indefinitely).
- **DVB CA module access**: `/dev/dvb/adapter<N>/ca<N>` requires CAP_SYS_RAWIO (smartcard PIN flows through this).
- **CEC PHYS_ADDR/LOG_ADDR sanitization**: every CEC_ADAP_S_LOG_ADDRS / S_PHYS_ADDR validated against EDID-derived PA tree; rogue compositor cannot claim arbitrary logical addresses.
- **dma-buf import for video LSM mediation**: cross-process video dma-buf import goes through file-LSM hook on the source-fd; cross-VM (vfio) import additionally validates IOMMU consistency.
- **VBI vbi-data scrubbing**: VBI data from analog tuners may contain captioning/teletext that exposes private info; per-channel allowlist optional (CONFIG_V4L2_VBI_ALLOWLIST).
- **Userspace-supplied USERPTR memory**: V4L2_MEMORY_USERPTR mode validated for page-pin lifetime via get_user_pages; userptr release guaranteed on STREAMOFF/dqbuf-cancel.

