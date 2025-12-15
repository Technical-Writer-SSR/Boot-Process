# Boot Process

![NXP Logo](https://img.shields.io/badge/NXP-Arm%20Processors-red)
![Linux Kernel](https://img.shields.io/badge/Linux-Kernel-green)
![Android](https://img.shields.io/badge/Android-Platform-brightgreen)
![ARM Architecture](https://img.shields.io/badge/ARM-Architecture-blue)
![U-Boot](https://img.shields.io/badge/Bootloader-U--Boot-yellow)
![Secure Boot](https://img.shields.io/badge/Secure-Boot-important)

## Overview
This document details the boot process for NXP ARM-based development boards running Linux and Android operating systems. The boot sequence follows a multi-stage architecture from power-on to full OS initialization.

## Boot Process Architecture

```mermaid
graph TD
    A[Power On/Reset] --> B[Boot ROM<br/>On-Chip ROM];
    B --> C{Boot Mode Selection<br/>eMMC/SD/USB/UART};
    
    subgraph "i.MX 8 Series"
        C --> D1[i.MX 8 Boot Flow];
        D1 --> E1[SCFW → SPL → ATF];
        E1 --> F1[U-Boot → Linux/Android];
    end
    
    subgraph "i.MX 6/7 Series"
        C --> D2[i.MX 6/7 Boot Flow];
        D2 --> E2[DCD → SPL → U-Boot];
        E2 --> F2[Linux Kernel];
    end
    
    subgraph "Layerscape Series"
        C --> D3[Layerscape Boot Flow];
        D3 --> E3[RCW → U-Boot → PPA];
        E3 --> F3[Linux/DPDK];
    end
    
    subgraph "Kinetis Series"
        C --> D4[Kinetis Boot Flow];
        D4 --> E4[XIP/MCUBoot];
        E4 --> F4[RTOS/Embedded];
    end
    
    F1 --> G[Device Tree Init];
    F2 --> G;
    F3 --> G;
    F4 --> G;
    
    G --> H[Kernel Initialization];
    H --> I[Init Process];
    I --> J[Userspace];
    
    style A fill:#f9f,stroke:#333,stroke-width:2px
    style J fill:#ccf,stroke:#333,stroke-width:2px
    style D1 fill:#d4f7d4
    style D2 fill:#d4e6f7
    style D3 fill:#f7d4e6
    style D4 fill:#f7f2d4
```

## Board-Specific Boot Flows

### 1. **i.MX 8 Series (i.MX 8M/8M Mini/8M Plus/8QM)**
```mermaid
graph LR
    A[Boot ROM] --> B[Load SCFW<br/>System Controller];
    B --> C[SCFW loads SPL];
    C --> D[SPL loads ATF<br/>ARM Trusted Firmware];
    D --> E[ATF loads U-Boot];
    E --> F{U-Boot Actions};
    F --> G[Load DTB];
    F --> H[Load Kernel];
    F --> I[AVB Verification];
    G & H & I --> J[Boot Linux/Android];
```

**Key Components:**
- **SCFW**: System Controller Firmware (manages power, clocks, security)
- **ATF**: ARM Trusted Firmware (EL3 secure monitor)
- **OPTEE**: Optional TEE for secure applications
- **AVB**: Android Verified Boot (for Android images)

**Typical Boards:** i.MX 8M EVK, i.MX 8M Mini EVK, i.MX 8M Plus EVK

### 2. **i.MX 6/7 Series (i.MX 6Quad/6Dual/6Solo/7Dual)**
```mermaid
graph LR
    A[Boot ROM] --> B[Read DCD<br/>Device Config Data];
    B --> C[Initialize DDR];
    C --> D[Load SPL<br/>Secondary Program Loader];
    D --> E[SPL loads U-Boot];
    E --> F{U-Boot Phase};
    F --> G[Setup Board];
    F --> H[Load Kernel];
    F --> I[Load DTB];
    G & H & I --> J[Boot Linux];
```

**Key Components:**
- **DCD**: Device Configuration Data (DDR timing parameters)
- **IVT**: Image Vector Table (image header structure)
- **HAB**: High Assurance Boot (security enforcement)

**Typical Boards:** Sabre SD, Sabre AI, i.MX 7Dual SABRE

### 3. **Layerscape Series (LX2160A, LS1046A, LS1028A)**
```mermaid
graph LR
    A[Power On] --> B[Load RCW<br/>Reset Configuration Word];
    B --> C[Initialize SerDes];
    C --> D[Load U-Boot];
    D --> E[Load PPA<br/>Platform Partition];
    E --> F[Setup DPC<br/>DPAA2 Complex];
    F --> G[Load Linux/DPDK];
    G --> H[Run Applications];
```

**Key Components:**
- **RCW**: Reset Configuration Word (pin multiplexing, SerDes config)
- **PPA**: Platform Partition Architecture (firmware partition)
- **DPAA2**: Data Path Acceleration Architecture

**Typical Boards:** Layerscape LX2160ARDB, LS1046ARDB, LS1028ARDB

### 4. **Kinetis Series (K64, K66, K28)**
```mermaid
graph LR
    A[Reset] --> B[Boot ROM];
    B --> C{Flash Type};
    C --> D[XIP Execution];
    C --> E[Load to RAM];
    D & E --> F[MCUBoot<br/>Optional];
    F --> G[RTOS/Embedded App];
    G --> H[Run Application];
```

**Key Components:**
- **XIP**: Execute-In-Place (run directly from flash)
- **MCUBoot**: Secure bootloader for MCUs
- **FlexSPI**: External flash interface

**Typical Boards:** FRDM-K64F, FRDM-K66F, TWR-K28F

## Common Boot Components

### Boot Media Options
| Media Type | Typical Use | Speed | Capacity |
|------------|-------------|-------|----------|
| eMMC | Primary storage | High | 4GB-64GB |
| SD Card | Development | Medium | Up to 512GB |
| QSPI Flash | Boot device | Medium | 16MB-256MB |
| NAND Flash | Cost-sensitive | Medium | 256MB-2GB |
| USB | Recovery/Download | Variable | N/A |

### Security Features
- **HAB** (High Assurance Boot): Code signing and verification
- **CAAM** (Cryptographic Accelerator): Hardware crypto engine
- **TrustZone**: ARM security extension
- **AVB 2.0**: Android Verified Boot
- **Secure JTAG**: Debug port protection

### Boot Configuration
```bash
# Typical U-Boot environment variables
bootcmd=mmc dev ${mmcdev}; mmc read ${loadaddr} ${kernel_addr} ${kernel_size}; \
        mmc read ${fdt_addr} ${fdt_addr_r} ${fdt_size}; \
        bootz ${loadaddr} - ${fdt_addr}

bootargs=console=ttymxc0,115200 root=/dev/mmcblk1p2 rootwait rw
```

## Quick Reference

### Boot Mode Selection
| Boot Pins | Mode | Description |
|-----------|------|-------------|
| 0000 | Serial Downloader | USB/UART recovery |
| 0001 | Internal Boot | Normal boot from flash |
| 0010 | USB Boot | Boot from USB device |
| 0100 | SD Card | Boot from SD/MMC |

### Common Commands
```bash
# U-Boot commands for i.MX
=> mmc list          # List MMC devices
=> fatls mmc 0:1     # List files on FAT partition
=> boot              # Boot with default settings
=> run netboot       # Network boot
=> fastboot          # Enter fastboot mode
```

## Troubleshooting

### Common Issues
1. **Boot hangs at SPL**: Check DDR initialization parameters
2. **HAB authentication fails**: Verify image signing
3. **Kernel panic early**: Check device tree compatibility
4. **No boot media detected**: Verify boot mode pins

### Debug Tools
- **UUU (Universal Update Utility)**: Flashing and recovery
- **OpenOCD**: JTAG debugging
- **U-Boot console**: Serial debug at 115200 baud
- **Linux dmesg**: Kernel boot messages

## Resources
- [NXP Official Documentation](https://www.nxp.com)
- [i.MX Linux Reference Manual](https://www.nxp.com/docs/en/user-guide/IMX_LINUX_USERS_GUIDE.pdf)
- [U-Boot Documentation](https://www.denx.de/wiki/U-Boot/Documentation)
- [ARM Trusted Firmware](https://trustedfirmware.org/)

## License
This documentation is provided for reference purposes. Refer to NXP official documentation for production use.

---
*Last Updated: *  
*For NXP i.MX, Layerscape, and Kinetis Platforms*
