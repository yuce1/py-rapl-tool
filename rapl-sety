#!/usr/bin/env python

import os
import sys
import struct
import math
import argparse
from terminaltables import AsciiTable

############## INTERNAL VARIABLES ##############
# 分别指代内核数量、socket数量以及每个socket的内核数量
num_core = 0
num_skt = 0
num_core_skt = 0
cpu_core = {}
power_lanes = ["pkg_limit_1", "pkg_limit_2", "dram", "pp0", "pp1"]

############## ARCHITECTURE INFO ##############
# 定义了三个unit,同一单位此处表示W，J，S
power_unit = None
energy_unit = None
time_unit = None

# 定义了msr寄存器中的数值约束
pkg_thermal_spec_power = None
pkg_minimum_power = None
pkg_maximum_power = None
pkg_maximum_time_windows = None

dram_thermal_spec_power = None
dram_minimum_power = None
dram_maximum_power = None
dram_maximum_time_windows = None

pp0_priority_level = None

pp1_priority_level = None

############## MSR ADDRESSES ##############

# 定义寄存器地址
MSR_RAPL_POWER_UNIT = 0x606

MSR_PKG_POWER_LIMIT = 0x610
MSR_PKG_ENERGY_STATUS = 0x611
MSR_PKG_PERF_STATUS = 0x613
MSR_PKG_POWER_INFO = 0x614

MSR_DRAM_POWER_LIMIT = 0x618
MSR_DRAM_ENERGY_STATUS = 0x619
MSR_DRAM_PERF_STATUS = 0x61b
MSR_DRAM_POWER_INFO = 0x61c

# Only used in Xeon architecture (PP0 -> core)
MSR_PP0_POWER_LIMIT = 0x638
MSR_PP0_ENERGY_STATUS = 0x639
MSR_PP0_POLICY   = 0x63a
MSR_PP0_PERF_STATUS = 0x63b

# Not used in Xeon architecture (PP1 -> graphic)
MSR_PP1_POWER_LIMIT = 0x640
MSR_PP1_ENERGY_STATUS = 0x641
MSR_PP1_POLICY = 0x642

##################################################

# 因为在寄存器中有最小单位，此处进行四舍五入计算
def my_round(n, base):
    err = n % base
    min_round = n - err
    max_rount = min_round + base

    if (max_rount - n) < (n - min_round):
        return max_rount
    else:
        return min_round

# Read a MSR registry
# 调用处都传了地址或者skt
# 如果找到了匹配项，那么后面的就不会再执行了
# 返回从指定的cpu的msr中读取的8个字节的值
def read_msr(msr, skt=None):
    cpu = None
    # 下面确定cpu得值，三选一
    if (skt is None):
        cpu = 0
        cpu = cpu + 10
    else:
        cpu = cpu_core.get(skt, 0)
        cpu = cpu + 10
    try:
        f = os.open('/dev/cpu/%d/msr' % (cpu,), os.O_RDONLY)
    except:
        sys.stderr.write("[ERROR] Failed to open /dev/cpu/%d/msr!\n"  % (cpu))
        exit(-1)

    os.lseek(f, msr, os.SEEK_SET)

    try:
        val = struct.unpack('Q', os.read(f, 8))[0]
    except:
        val = None

    os.close(f)

    return val

# Write a MSR registry
# 比上边的方法多了一个参数 value的值，长度为8字节
def write_msr(msr, val, skt=None):
    cpu = None
    # 下面确定cpu得值，三选一
    if (skt is None):
        cpu = 0
        cpu = cpu + 10
    else:
        cpu = cpu_core.get(skt, 0)
        cpu = cpu + 10
    try:
        f = os.open('/dev/cpu/%d/msr' % (cpu,), os.O_WRONLY)
    except:
        sys.stderr.write("[ERROR] Failed to open /dev/cpu/%d/msr!\n"  % (cpu))
        exit(-1)

    os.lseek(f, msr, os.SEEK_SET)

    try:
        os.write(f, struct.pack('Q', val))
    except:
        pass

    os.close(f)

# Check requisites of the system
# 进行运行环境的检查
def check_requirements():
    # Check root permission
    uid = os.getuid()
    if uid != 0:
        sys.stderr.write("[WARNING] Need to be root to execute this software!\n")
        exit(-1)

    # Check if MSR module is loaded
    if not os.path.exists("/dev/cpu/0/msr"):
        sys.stderr.write("[WARNING] MSR module is not loaded!\n")
        exit(-2)

# Default initial configuration
# 对以下信息进行初始化
def init_config():
    global num_core, num_skt, num_core_skt, cpu_core
    global init_rapl_config, nominal_frequency, power_unit, energy_unit, time_unit
    global pkg_thermal_spec_power, pkg_minimum_power, pkg_maximum_power, pkg_maximum_time_windows
    global dram_thermal_spec_power, dram_minimum_power, dram_maximum_power, dram_maximum_time_windows

    # Set affinity on a specific core
    #core = 0
    #os.system("taskset -cp %d %d >/dev/null 2>&1" % (core, os.getpid()))

    # Read the number of virtual CPUs and sockets
    # 本机的运行环境中获取到的是逻辑cpu 80  和真实的物理内核 2

    with open("/proc/cpuinfo", "r") as f:
        conf_lines = f.readlines()
        physical_id = -1
        processor = -1
        for cl in conf_lines:
            # 记录core id
            if "processor" in cl:
                processor = cl.replace(" ", "").replace('\n','').replace('\r','').split(":")[1]
                processor = int(processor)
            # 记录physical id
            if "physical id" in cl:
                num_core += 1
                physical_id = cl.replace(" ", "").replace('\n','').replace('\r','').split(":")[1]
                physical_id = int(physical_id)
            if(physical_id != -1 and processor != -1):
                if(physical_id in cpu_core):
                    physical_id = -1
                    processor = -1
                else:
                    cpu_core[physical_id] = processor
                    physical_id = -1
                    processor = -1
    num_skt = len(cpu_core)
    num_core_skt = num_core / num_skt
    rapl_power_unit = read_msr(MSR_RAPL_POWER_UNIT)
    pkg_power_info = read_msr(MSR_PKG_POWER_INFO)
    dram_power_info = read_msr(MSR_DRAM_POWER_INFO)

    # Unit conversions specified in the RAPL chapter of Intel manual
    # 获取unit
    if rapl_power_unit is not None:
        power_unit = pow(0.5, rapl_power_unit & 0xF)
        energy_unit = pow(0.5, (rapl_power_unit >> 8) & 0x1F)
        time_unit = pow(0.5, (rapl_power_unit >> 16) & 0xF)

    if rapl_power_unit is not None and pkg_power_info is not None:
        pkg_thermal_spec_power = (pkg_power_info & 0x7FFF) * power_unit
        pkg_minimum_power = ((pkg_power_info >> 16) & 0x7FFF) * power_unit
        pkg_maximum_power = ((pkg_power_info >> 32) & 0x7FFF) * power_unit
        pkg_maximum_time_windows = ((pkg_power_info >> 48) & 0x3F) * time_unit

    if rapl_power_unit is not None and dram_power_info is not None:
        dram_thermal_spec_power = (dram_power_info & 0x7FFF) * power_unit
        dram_minimum_power = ((dram_power_info >> 16) & 0x7FFF) * power_unit
        dram_maximum_power = ((dram_power_info >> 32) & 0x7FFF) * power_unit
        dram_maximum_time_windows = ((dram_power_info >> 48) & 0x3F) * time_unit

# Read package limit 1 RAPL
def read_pkg_limit_1_rapl(skt):
    msr_rapl = read_msr(MSR_PKG_POWER_LIMIT, skt)
    if msr_rapl == None:
        return None, None, None, None, None

    power_limit = (msr_rapl & 0x7FFF) * power_unit * 1000000
    enable = True if (msr_rapl >> 15) & 0x1 else False
    clamping = True if (msr_rapl >> 16) & 0x1 else False
    time_limit_temp = int((msr_rapl >> 17) & 0x7F)
    Y = (time_limit_temp & 0x1F)
    Z = ((time_limit_temp >> 5) & 0x3)
    time_limit = math.pow(2 , Y) * (1.0 + Z / 4.0) * time_unit * 1000000
    lock = True if (msr_rapl >> 63) & 0x1 else False

    return power_limit, enable, clamping, time_limit, lock

# Read package limit 2 RAPL
def read_pkg_limit_2_rapl(skt):
    msr_rapl = read_msr(MSR_PKG_POWER_LIMIT, skt)
    if msr_rapl == None:
        return None, None, None, None, None

    msr_rapl = (msr_rapl >> 32)

    power_limit = (msr_rapl & 0x7FFF) * power_unit * 1000000
    enable = True if (msr_rapl >> 15) & 0x1 else False
    clamping = True if (msr_rapl >> 16) & 0x1 else False
    time_limit_temp = ((msr_rapl >> 17) & 0x7F)
    Y = (time_limit_temp & 0x1F)
    Z = ((time_limit_temp >> 5) & 0x3)
    time_limit = math.pow(2 , Y) * (1.0 + Z / 4.0) * time_unit * 1000000
    lock = True if (msr_rapl >> 31) & 0x1 else False

    return power_limit, enable, clamping, time_limit, lock

# Read dram RAPL
def read_dram_rapl(skt):
    msr_rapl = read_msr(MSR_DRAM_POWER_LIMIT, skt)
    if msr_rapl == None:
        return None, None, None, None, None

    power_limit = (msr_rapl & 0x7FFF) * power_unit * 1000000
    enable = True if (msr_rapl >> 15) & 0x1 else False
    clamping = True if (msr_rapl >> 16) & 0x1 else False
    time_limit_temp = ((msr_rapl >> 17) & 0x7F)
    Y = (time_limit_temp & 0x1F)
    F = ((time_limit_temp >> 5) & 0x3)
    time_limit = math.pow(2 , Y) * (1.0 + F / 10.0) * time_unit * 1000000
    lock = True if (msr_rapl >> 31) & 0x1 else False

    return power_limit, enable, clamping, time_limit, lock

# Read PP0 RAPL
def read_pp0_rapl(skt):
    msr_rapl = read_msr(MSR_PP0_POWER_LIMIT, skt)
    if msr_rapl == None:
        return None, None, None, None, None, None

    power_limit = (msr_rapl & 0x7FFF) * power_unit * 1000000
    enable = True if (msr_rapl >> 15) & 0x1 else False
    clamping = True if (msr_rapl >> 16) & 0x1 else False
    time_limit_temp = ((msr_rapl >> 17) & 0x7F)
    Y = (time_limit_temp & 0x1F)
    F = ((time_limit_temp >> 5) & 0x3)
    time_limit = math.pow(2 , Y) * (1.0 + F / 10.0) * time_unit * 1000000
    lock = True if (msr_rapl >> 31) & 0x1 else False

    msr_rapl = read_msr(MSR_PP0_POLICY, skt)
    if msr_rapl == None:
        return power_limit, enable, clamping, time_limit, lock, None
    else:
        priority_level = msr_rapl & 0x1F
        return power_limit, enable, clamping, time_limit, lock, priority_level

# Read PP1 RAPL
def read_pp1_rapl(skt):
    msr_rapl = read_msr(MSR_PP1_POWER_LIMIT, skt)
    if msr_rapl == None:
        return None, None, None, None, None, None

    power_limit = (msr_rapl & 0x7FFF) * power_unit * 1000000
    enable = True if (msr_rapl >> 15) & 0x1 else False
    clamping = True if (msr_rapl >> 16) & 0x1 else False
    time_limit_temp = ((msr_rapl >> 17) & 0x7F)
    Y = (time_limit_temp & 0x1F)
    F = ((time_limit_temp >> 5) & 0x3)
    time_limit = math.pow(2 , Y) * (1.0 + F / 10.0) * time_unit * 1000000
    lock = True if (msr_rapl >> 31) & 0x1 else False

    msr_rapl = read_msr(MSR_PP1_POLICY, skt)
    if msr_rapl == None:
        return power_limit, enable, clamping, time_limit, lock, None
    else:
        priority_level = msr_rapl & 0x1F
        return power_limit, enable, clamping, time_limit, lock, priority_level

def read_pkg_pc_energy(skt):
    msr_rapl = read_msr(MSR_PKG_ENERGY_STATUS, skt)
    if msr_rapl == None:
        return None

    return (msr_rapl & 0xFFFFFFFF) * energy_unit

def read_pkg_pc_throttled_time(skt):
    msr_rapl = read_msr(MSR_PKG_PERF_STATUS, skt)
    if msr_rapl == None:
        return None

    return (msr_rapl & 0xFFFFFFFF) * time_unit

def read_dram_pc_energy(skt):
    msr_rapl = read_msr(MSR_DRAM_ENERGY_STATUS, skt)
    if msr_rapl == None:
        return None

    return (msr_rapl & 0xFFFFFFFF) * energy_unit

def read_dram_pc_throttled_time(skt):
    msr_rapl = read_msr(MSR_DRAM_PERF_STATUS, skt)
    if msr_rapl == None:
        return None

    return (msr_rapl & 0xFFFFFFFF) * time_unit

def read_pp0_pc_energy(skt):
    msr_rapl = read_msr(MSR_PP0_ENERGY_STATUS, skt)
    if msr_rapl == None:
        return None

    return (msr_rapl & 0xFFFFFFFF) * energy_unit

def read_pp0_pc_throttled_time(skt):
    msr_rapl = read_msr(MSR_PP0_PERF_STATUS, skt)
    if msr_rapl == None:
        return None

    return (msr_rapl & 0xFFFFFFFF) * time_unit

def read_pp1_pc_energy(skt):
    msr_rapl = read_msr(MSR_PP1_ENERGY_STATUS, skt)
    if msr_rapl == None:
        return None

    return (msr_rapl & 0xFFFFFFFF) * energy_unit

# Set RAPL power limit in Watts
# 传入单位是 微瓦
def set_power_limit(skt, power_lane, power_limit):
    # Convert to RAPL unit
    # 转换成瓦
    power_limit_sec = power_limit / 1000000.0
    power_cap_rapl = int(my_round(power_limit_sec, power_unit) / power_unit)

    # Set only bits related to power limit
    if "pkg_limit_1" in power_lane or "pkg_limit_2" in power_lane:
        msr_rapl = read_msr(MSR_PKG_POWER_LIMIT, skt)
    elif "dram" in power_lane:
        msr_rapl = read_msr(MSR_DRAM_POWER_LIMIT, skt)
    elif "pp0" in power_lane:
        msr_rapl = read_msr(MSR_PP0_POWER_LIMIT, skt)
    elif "pp1" in power_lane:
        msr_rapl = read_msr(MSR_PP1_POWER_LIMIT, skt)

    if msr_rapl is None:
        return

    if "pkg_limit_2" in power_lane:
        msr_rapl = (msr_rapl & ~0x7FFF00000000) | (power_cap_rapl << 32)
    else:
        msr_rapl = (msr_rapl & ~0x7FFF) | power_cap_rapl

    if "pkg_limit_1" in power_lane or "pkg_limit_2" in power_lane:
        write_msr(MSR_PKG_POWER_LIMIT, msr_rapl, skt)
    elif "dram" in power_lane:
        write_msr(MSR_DRAM_POWER_LIMIT, msr_rapl, skt)
    elif "pp0" in power_lane:
        write_msr(MSR_PP0_POWER_LIMIT, msr_rapl, skt)
    elif "pp1" in power_lane:
        write_msr(MSR_PP1_POWER_LIMIT, msr_rapl, skt)

# Set RAPL time window in ms
# 传入单位是微秒
def set_time_window(skt, power_lane, time_window):
    # Convert to RAPL unit
    time_window_sec = time_window / 1000000.0
    time_window_rapl = int(my_round(time_window_sec, time_unit) / time_unit)


    # Set only bits related to power limit
    if "pkg_limit_1" in power_lane or "pkg_limit_2" in power_lane:
        msr_rapl = read_msr(MSR_PKG_POWER_LIMIT, skt)
    elif "dram" in power_lane:
        msr_rapl = read_msr(MSR_DRAM_POWER_LIMIT, skt)
    elif "pp0" in power_lane:
        msr_rapl = read_msr(MSR_PP0_POWER_LIMIT, skt)
    elif "pp1" in power_lane:
        msr_rapl = read_msr(MSR_PP1_POWER_LIMIT, skt)

    if msr_rapl is None:
        return

    # 进行时间处理，不同的域有不同的时间
    if "pkg_limit_2" in power_lane:
        i = 0
        while time_window_rapl > math.pow(2 , i):
            i = i + 1
        Y = i - 1
        Z = int(((time_window_rapl / (math.pow(2,Y))) - 1.0) * 4.0)
        msr_rapl = (msr_rapl & ~0xFE000000000000) | (Y << 49) | (Z << 54)
    elif "pkg_limit_1" in power_lane:
        i = 0
        while time_window_rapl > math.pow(2 , i):
            i = i + 1
        Y = i - 1
        Z = int(((time_window_rapl / (math.pow(2,Y))) - 1.0) * 4.0)
        msr_rapl = (msr_rapl & ~0xFE0000) | (Y << 17) | (Z << 22)
    else:
        i = 0
        while time_window_rapl > math.pow(2 , i):
            i = i + 1
        Y = i - 1
        F = int(((time_window_rapl / (math.pow(2,Y))) - 1.0) * 10.0)
        msr_rapl = (msr_rapl & ~0xFE0000) | (Y << 17) | (F << 22)

    if "pkg_limit_1" in power_lane or "pkg_limit_2" in power_lane:
        write_msr(MSR_PKG_POWER_LIMIT, msr_rapl, skt)
    elif "dram" in power_lane:
        write_msr(MSR_DRAM_POWER_LIMIT, msr_rapl, skt)
    elif "pp0" in power_lane:
        write_msr(MSR_PP0_POWER_LIMIT, msr_rapl, skt)
    elif "pp1" in power_lane:
        write_msr(MSR_PP1_POWER_LIMIT, msr_rapl, skt)

# Set RAPL enable limit
def set_enable_limit(skt, power_lane, setting):
    # Set only bits related to power limit
    if "pkg_limit_1" in power_lane or "pkg_limit_2" in power_lane:
        msr_rapl = read_msr(MSR_PKG_POWER_LIMIT, skt)
    elif "dram" in power_lane:
        msr_rapl = read_msr(MSR_DRAM_POWER_LIMIT, skt)
    elif "pp0" in power_lane:
        msr_rapl = read_msr(MSR_PP0_POWER_LIMIT, skt)
    elif "pp1" in power_lane:
        msr_rapl = read_msr(MSR_PP1_POWER_LIMIT, skt)

    if msr_rapl is None:
        return

    setting_rapl = 0x1 if setting is True else 0x0

    if "pkg_limit_2" in power_lane:
        msr_rapl = (msr_rapl & ~0x800000000000) | (setting_rapl << 47)
    else:
        msr_rapl = (msr_rapl & ~0x8000) | (setting_rapl << 15)

    if "pkg_limit_1" in power_lane or "pkg_limit_2" in power_lane:
        write_msr(MSR_PKG_POWER_LIMIT, msr_rapl, skt)
    elif "dram" in power_lane:
        write_msr(MSR_DRAM_POWER_LIMIT, msr_rapl, skt)
    elif "pp0" in power_lane:
        write_msr(MSR_PP0_POWER_LIMIT, msr_rapl, skt)
    elif "pp1" in power_lane:
        write_msr(MSR_PP1_POWER_LIMIT, msr_rapl, skt)

# Set RAPL clamping limit
def set_clamping_limit(skt, power_lane, setting):
    # Set only bits related to power limit
    msr_rapl = None
    if "pkg_limit_1" in power_lane or "pkg_limit_2" in power_lane:
        msr_rapl = read_msr(MSR_PKG_POWER_LIMIT, skt)
    elif "pp0" in power_lane:
        msr_rapl = read_msr(MSR_PP0_POWER_LIMIT, skt)
    elif "pp1" in power_lane:
        msr_rapl = read_msr(MSR_PP1_POWER_LIMIT, skt)

    if msr_rapl is None:
        return

    setting_rapl = 0x1 if setting is True else 0x0

    if "pkg_limit_2" in power_lane:
        msr_rapl = (msr_rapl & ~0x1000000000000) | (setting_rapl << 48)
    else:
        msr_rapl = (msr_rapl & ~0x10000) | (setting_rapl << 16)

    if "pkg_limit_1" in power_lane or "pkg_limit_2" in power_lane:
        write_msr(MSR_PKG_POWER_LIMIT, msr_rapl, skt)
    elif "pp0" in power_lane:
        write_msr(MSR_PP0_POWER_LIMIT, msr_rapl, skt)
    elif "pp1" in power_lane:
        write_msr(MSR_PP1_POWER_LIMIT, msr_rapl, skt)

# Set RAPL lock
def set_lock(skt, power_lane):
    # Set only bits related to power limit
    msr_rapl = None
    if "pkg_limit_1" in power_lane or "pkg_limit_2" in power_lane:
        msr_rapl = read_msr(MSR_PKG_POWER_LIMIT, skt)
    elif "dram" in power_lane:
        msr_rapl = read_msr(MSR_DRAM_POWER_LIMIT, skt)
    elif "pp0" in power_lane:
        msr_rapl = read_msr(MSR_PP0_POWER_LIMIT, skt)
    elif "pp1" in power_lane:
        msr_rapl = read_msr(MSR_PP1_POWER_LIMIT, skt)

    if msr_rapl is None:
        return

    if "pkg_limit_1" in power_lane or "pkg_limit_2" in power_lane:
        msr_rapl = (msr_rapl & ~0x8000000000000000) | (0x1 << 63)
    else:
        msr_rapl = (msr_rapl & ~0x80000000) | (0x1 << 31)

    if "pkg_limit_1" in power_lane or "pkg_limit_2" in power_lane:
        write_msr(MSR_PKG_POWER_LIMIT, msr_rapl, skt)
    elif "dram" in power_lane:
        write_msr(MSR_DRAM_POWER_LIMIT, msr_rapl, skt)
    elif "pp0" in power_lane:
        write_msr(MSR_PP0_POWER_LIMIT, msr_rapl, skt)
    elif "pp1" in power_lane:
        write_msr(MSR_PP1_POWER_LIMIT, msr_rapl, skt)

def set_priority_level(skt, power_lane, priority_level):
    # Set only bits related to power limit
    if "pp0" in power_lane:
        msr_rapl = read_msr(MSR_PP0_POLICY, skt)
    elif "pp1" in power_lane:
        msr_rapl = read_msr(MSR_PP1_POLICY, skt)

    if msr_rapl is None:
        return

    msr_rapl = (msr_rapl & ~0x1F) | priority_level

    if "pp0" in power_lane:
        write_msr(MSR_PP0_POLICY, msr_rapl, skt)
    elif "pp1" in power_lane:
        write_msr(MSR_PP1_POLICY, msr_rapl, skt)

def print_conf():
    # Configuration parameteres
    table_data = []

    socket_row = ["Conf Parameter"]
    for s in range(num_skt):
        socket_row.append("Package " + str(s))
    table_data.append(socket_row)

    pkg_power_limit_1 = ["PKG_limit_1 power limit (uW)"]
    pkg_enable_1 = ["PKG_limit_1 enable limit"]
    pkg_clamping_1 = ["PKG_limit_1 clamping limit"]
    pkg_time_limit_1 = ["PKG_limit_1 time window (us)"]
    pkg_lock_1 = ["PKG_limit_1 lock"]
    for s in range(num_skt):
        power_limit, enable, clamping, time_limit, lock = read_pkg_limit_1_rapl(s)
        pkg_power_limit_1.append(power_limit if power_limit is not None else "N/A")
        pkg_enable_1.append(enable if enable is not None else "N/A")
        pkg_clamping_1.append(clamping if clamping is not None else "N/A")
        pkg_time_limit_1.append(time_limit if time_limit is not None else "N/A")
        pkg_lock_1.append(lock if lock is not None else "N/A")
    table_data += [pkg_power_limit_1, pkg_time_limit_1, pkg_enable_1, pkg_clamping_1, pkg_lock_1]

    pkg_power_limit_2 = ["PKG_limit_2 power limit (uW)"]
    pkg_enable_2 = ["PKG_limit_2 enable limit"]
    pkg_clamping_2 = ["PKG_limit_2 clamping limit"]
    pkg_time_limit_2 = ["PKG_limit_2 time window (us)"]
    pkg_lock_2 = ["PKG_limit_2 lock"]
    for s in range(num_skt):
        power_limit, enable, clamping, time_limit, lock = read_pkg_limit_2_rapl(s)
        pkg_power_limit_2.append(power_limit if power_limit is not None else "N/A")
        pkg_enable_2.append(enable if enable is not None else "N/A")
        pkg_clamping_2.append(clamping if clamping is not None else "N/A")
        pkg_time_limit_2.append(time_limit if time_limit is not None else "N/A")
        pkg_lock_2.append(lock if lock is not None else "N/A")
    table_data += [pkg_power_limit_2, pkg_time_limit_2, pkg_enable_2, pkg_clamping_2, pkg_lock_2]

    dram_power_limit = ["DRAM power limit (uW)"]
    dram_enable = ["DRAM enable limit"]
    dram_time_limit = ["DRAM time window (us)"]
    dram_lock = ["DRAM lock"]
    for s in range(num_skt):
        power_limit, enable, clamping, time_limit, lock = read_dram_rapl(s)
        dram_power_limit.append(power_limit if power_limit is not None else "N/A")
        dram_enable.append(enable if enable is not None else "N/A")
        dram_time_limit.append(time_limit if time_limit is not None else "N/A")
        dram_lock.append(lock if lock is not None else "N/A")
    table_data += [dram_power_limit, dram_time_limit, dram_enable, dram_lock]

    pp0_power_limit = ["PP0 power limit (uW)"]
    pp0_enable = ["PP0 enable limit"]
    pp0_clamping = ["PP0 clamping limit"]
    pp0_time_limit = ["PP0 time window (us)"]
    pp0_priority_level = ["PP0 priority_level"]
    pp0_lock = ["PP0 lock"]
    for s in range(num_skt):
        power_limit, enable, clamping, time_limit, lock, priority_level = read_pp0_rapl(s)
        pp0_power_limit.append(power_limit if power_limit is not None else "N/A")
        pp0_enable.append(enable if enable is not None else "N/A")
        pp0_clamping.append(clamping if clamping is not None else "N/A")
        pp0_time_limit.append(time_limit if time_limit is not None else "N/A")
        pp0_lock.append(lock if lock is not None else "N/A")
        pp0_priority_level.append(priority_level if priority_level is not None else "N/A")
    table_data += [pp0_power_limit, pp0_time_limit, pp0_enable, pp0_clamping, pp0_priority_level, pp0_lock]

    pp1_power_limit = ["PP1 power limit (uW)"]
    pp1_enable = ["PP1 enable limit"]
    pp1_clamping = ["PP1 clamping limit"]
    pp1_time_limit = ["PP1 time window (us)"]
    pp1_priority_level = ["PP1 priority_level"]
    pp1_lock = ["PP1 lock"]
    for s in range(num_skt):
        power_limit, enable, clamping, time_limit, lock, priority_level = read_pp1_rapl(s)
        pp1_power_limit.append(power_limit if power_limit is not None else "N/A")
        pp1_enable.append(enable if enable is not None else "N/A")
        pp1_clamping.append(clamping if clamping is not None else "N/A")
        pp1_time_limit.append(time_limit if time_limit is not None else "N/A")
        pp1_lock.append(lock if lock is not None else "N/A")
        pp1_priority_level.append(priority_level if priority_level is not None else "N/A")
    table_data += [pp1_power_limit, pp1_time_limit, pp1_enable, pp1_clamping, pp1_priority_level, pp1_lock]

    table = AsciiTable(table_data)
    print (table.table)

    # Architectural parameteres
    table_data = []

    socket_row = ["Arch Parameter", "Value"]
    table_data.append(socket_row)

    table_data.append(["Time units (us)", time_unit * 1000000])
    table_data.append(["Power units (uW)", power_unit * 1000000])
    table_data.append(["Energy units (uJ)", energy_unit * 1000000])

    table_data.append(["PKG Thermal Spec Power (uW)", pkg_thermal_spec_power * 1000000 if pkg_thermal_spec_power is not None else "N/A"])
    table_data.append(["PKG Minimum Power (uW)", pkg_minimum_power * 1000000 if pkg_minimum_power is not None else "N/A"])
    table_data.append(["PKG Maximum Power (uW)", pkg_maximum_power * 1000000 if pkg_maximum_power is not None else "N/A"])
    table_data.append(["PKG Maximum Time Window (us)", pkg_maximum_time_windows * 1000000 if pkg_maximum_time_windows is not None else "N/A"])

    table_data.append(["DRAM Thermal Spec Power (uW)", dram_thermal_spec_power * 1000000 if dram_thermal_spec_power is not None else "N/A"])
    table_data.append(["DRAM Minimum Power (uW)", dram_minimum_power * 1000000 if dram_minimum_power is not None else "N/A"])
    table_data.append(["DRAM Maximum Power (uW)", dram_maximum_power * 1000000 if dram_maximum_power is not None else "N/A"])
    table_data.append(["DRAM Maximum Time Window (us)", dram_maximum_time_windows * 1000000 if dram_maximum_time_windows is not None else "N/A"])

    table = AsciiTable(table_data)
    print (table.table)

    # Performance counters
    table_data = []

    socket_row = ["Perf counter"]
    for s in range(num_skt):
        socket_row.append("Socket " + str(s))
    table_data.append(socket_row)

    pkg_energy = ["PKG energy (uJ)"]
    pkg_throttled = ["PKG throttled time (us)"]
    dram_energy = ["DRAM energy (uJ)"]
    dram_throttled = ["DRAM throttled time (us)"]
    pp0_energy = ["PP0 energy (uJ)"]
    pp0_throttled = ["PP0 throttled time (us)"]
    pp1_energy = ["PP1 energy (uJ)"]
    for s in range(num_skt):
        pkg_pc_energy = read_pkg_pc_energy(s)
        pkg_energy.append(pkg_pc_energy * 1000000 if pkg_pc_energy is not None else "N/A")

        pkg_pc_throttled_time = read_pkg_pc_throttled_time(s)
        pkg_throttled.append(pkg_pc_throttled_time  * 1000000 if pkg_pc_throttled_time is not None else "N/A")

        dram_pc_energy = read_dram_pc_energy(s)
        dram_energy.append(dram_pc_energy  * 1000000 if dram_pc_energy is not None else "N/A")

        dram_pc_throttled_time = read_dram_pc_throttled_time(s)
        dram_throttled.append(dram_pc_throttled_time * 1000000  if dram_pc_throttled_time is not None else "N/A")

        pp0_pc_energy = read_pp0_pc_energy(s)
        pp0_energy.append(pp0_pc_energy  * 1000000 if pp0_pc_energy is not None else "N/A")

        pp0_pc_throttled_time = read_pp0_pc_throttled_time(s)
        pp0_throttled.append(pp0_pc_throttled_time  * 1000000 if pp0_pc_throttled_time is not None else "N/A")

        pp1_pc_energy = read_pp1_pc_energy(s)
        pp1_energy.append(pp1_pc_energy  * 1000000 if pp1_pc_energy is not None else "N/A")

    table_data += [pkg_energy, pkg_throttled, dram_energy, dram_throttled, pp0_energy, pp0_throttled, pp1_energy]

    table = AsciiTable(table_data)
    print (table.table)

if __name__ == "__main__":
    check_requirements()

    init_config()

    parser = argparse.ArgumentParser(description='Tool for RAPL setting')
    parser.add_argument('-s','--socket', help='Select the socket (default: all the sockets)', required=False, type=int, choices=range(0, num_skt))
    parser.add_argument('-p','--power-lane', help='Select the power lane', required=False, choices=power_lanes)
    parser.add_argument('-l','--power-limit', help='Set power limit (uW)', required=False, type=float)
    parser.add_argument('-t','--time-window', help='Set time window (us)', required=False, type=float)
    parser.add_argument('-el','--enable-limit', help='Force enable to power limit', required=False, action='store_true', default=False)
    parser.add_argument('-dl','--disable-limit', help='Force disable to power limit', required=False, action='store_true', default=False)
    parser.add_argument('-ec','--enable-clamping', help='Force enable to clamping limit', required=False, action='store_true', default=False)
    parser.add_argument('-dc','--disable-clamping', help='Force disable to clamping limit', required=False, action='store_true', default=False)
    parser.add_argument('-sl','--set-lock', help='Set lock for power lane (WARNING: all write attempts are ignored until next RESET!)', required=False, action='store_true', default=False)
    parser.add_argument('-pl','--priority-level', help='Select the priority level', required=False, type=int, choices=range(17))
    args = parser.parse_args()

    if len(sys.argv) == 1:
        print_conf()
    else:
        if args.power_lane is not None:
            sockets = []

            if args.socket is not None:
                sockets.append(args.socket)
            else:
                sockets = [i for i in range(num_skt)]

            for skt in sockets:
                if args.power_limit is not None:
                    set_power_limit(skt, args.power_lane, args.power_limit)

                if args.time_window is not None:
                    set_time_window(skt, args.power_lane, args.time_window)

                if args.enable_limit is True:
                    set_enable_limit(skt, args.power_lane, True)
                elif args.disable_limit is True:
                    set_enable_limit(skt, args.power_lane, False)

                if args.enable_clamping is True:
                    set_clamping_limit(skt, args.power_lane, True)
                elif args.disable_clamping is True:
                    set_clamping_limit(skt, args.power_lane, False)

                if args.set_lock is True:
                    set_lock(skt, args.power_lane)

                if args.priority_level is not None:
                    set_priority_level(skt, args.power_lane, args.priority_level)
