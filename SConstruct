#!/usr/bin/python

import re
from glob import glob
import os

env = Environment()
platform = env['PLATFORM']

VARTAB = {}

def resolve_var(varname, default_value):
    global VARTAB
    # precedence: scons argument -> environment variable -> default value
    ret = ARGUMENTS.get(varname, None)
    VARTAB[varname] = ('arg', ret)
    if ret == None:
        ret = os.environ.get(varname, None)
        VARTAB[varname] = ('env', ret)
    if ret == None:
        ret = default_value
        VARTAB[varname] = ('dfl', ret)
    return ret

def getUsbTty(rx):
    usb_ttys = glob(rx)
    return usb_ttys[0] if len(usb_ttys) == 1 else None

ARDUINO_HOME = resolve_var('ARDUINO_HOME', '/home/uk/arduino-1.5.2')

ARDUINO_PORT = resolve_var('ARDUINO_PORT', '/dev/ttyACM0')
SKETCHBOOK_HOME = resolve_var('SKETCHBOOK_HOME',
                              os.path.expanduser('~/sketchbook'))

ARDUINO_PLATFORM = resolve_var('ARDUINO_PLATFORM', 'sam')
ARDUINO_BOARD = resolve_var('ARDUINO_BOARD', 'arduino_due_x')

ARDUINO_CORE_PATH = os.path.join(ARDUINO_HOME, 'hardware/arduino', ARDUINO_PLATFORM, 'cores/arduino')
ARDUINO_VARIANT_PATH = os.path.join(ARDUINO_HOME, 'hardware/arduino', ARDUINO_PLATFORM, 'variants', ARDUINO_BOARD)

ARDUINO_SKEL = os.path.join(ARDUINO_CORE_PATH, 'main.cpp')

ARDUINO_VER = resolve_var('ARDUINO_VER', '152')

RST_TRIGGER = resolve_var('RST_TRIGGER', None)
EXTRA_LIB = resolve_var('EXTRA_LIB', None)

ARDUINO_TOOLS_PATH = os.path.join(ARDUINO_HOME, 'hardware/tools')
ARDUINO_GCC_PATH = os.path.join(ARDUINO_TOOLS_PATH, 'g++_arm_none_eabi')
ARDUINO_GCC_BIN_PATH = os.path.join(ARDUINO_GCC_PATH, 'bin')
ARDUINO_GCC_BIN_PREFIX = 'arm-none-eabi-'

def arduino_gcc_bin(name):
    return os.path.join(ARDUINO_GCC_BIN_PATH, ARDUINO_GCC_BIN_PREFIX + name)

TARGET = os.path.basename(os.path.realpath(os.curdir))

CFLAGS = [
    '-Wall',
    '-Os',
    '-w',
    '-ffunction-sections',
    '-fdata-sections',
    '-nostdlib',
    '--param',
    'max-inline-insns-single=500',
    '-fno-rtti',
    '-fno-exceptions',
    '-mcpu=cortex-m3',
    '-mthumb',
]

CPPDEFINES = {
    'printf' : 'iprintf',
    'F_CPU' : '84000000L',
    'ARDUINO' : ARDUINO_VER,
    '__SAM3X8E__' : None,
    'USB_PID' : '0x003e',
    'USB_VID' : '0x2341',
    'USBCON' : None
}

CPPPATH = [
    ARDUINO_CORE_PATH,
    ARDUINO_VARIANT_PATH,
    os.path.join(ARDUINO_HOME, 'hardware/arduino/sam/system/libsam'),
    os.path.join(ARDUINO_HOME, 'hardware/arduino/sam/system/CMSIS/CMSIS/Include'),
    os.path.join(ARDUINO_HOME, 'hardware/arduino/sam/system/CMSIS/Device/ATMEL'),
]

LIBS = [
    'gcc',
    'm',
]

LINKFLAGS = [
    '-Os',
    '-mcpu=cortex-m3',
    '-mthumb',
    '-T%s/linker_scripts/gcc/flash.ld' % ARDUINO_VARIANT_PATH,
    '-Wl,-Map,/home/uk/sketch_feb13a.cpp.map',
    '-Wl,--cref',
    '-Wl,--check-sections',
    '-Wl,--gc-sections',
    '-Wl,--entry=Reset_Handler',
    '-Wl,--unresolved-symbols=report-all',
    '-Wl,--warn-common',
    '-Wl,--warn-section-align',
    '-Wl,--warn-unresolved-symbols',
    '-Wl,--start-group',
    'libproj.a',
    'libcore.a',
    env.File(os.path.join(ARDUINO_VARIANT_PATH, 'libsam_sam3x8e_gcc_rel.a')),
    '-Wl,--end-group',
]

def touch_port(baudrate):
    import serial
    import time

    print "Touch Port %s (%d)" % (ARDUINO_PORT, baudrate)

    port = serial.Serial(port=ARDUINO_PORT,
                         baudrate=baudrate,
                         bytesize=serial.EIGHTBITS,
                         parity=serial.PARITY_NONE,
                         stopbits=serial.STOPBITS_ONE)
    port.close()
    time.sleep(0.5)



def touch_port_1200(target, source, env):
    touch_port(1200)

def touch_port_9600(target, source, env):
    touch_port(9600)

env = Environment(CC = arduino_gcc_bin("gcc"),
                  CXX = arduino_gcc_bin("g++"),
                  AS = arduino_gcc_bin("as"),
                  CPPPATH = CPPPATH,
                  CPPDEFINES = CPPDEFINES,
                  CFLAGS = CFLAGS,
                  CCFLAGS = CFLAGS,
                  ASFLAGS = '',
                  LINKFLAGS = LINKFLAGS,
                  LIBS = LIBS,
)

def _rebase_sources(rebase, base, dirname):
    prog = re.compile('.+\.(c|cpp|S)$')
    path = os.path.join(base, dirname)
    for _, dirnames, filenames in os.walk(path, topdown=True):
        for filename in filenames:
            if prog.match(filename) is not None:
                yield os.path.join(rebase, dirname, filename)
        for subdirname in dirnames:
            for filename in _rebase_sources(rebase, base, os.path.join(dirname, subdirname)):
                yield filename
        del dirnames[:]

def rebase_sources(rebase, base, dirname=""):
    return list(_rebase_sources(rebase, base, dirname))

env.VariantDir('build/core', ARDUINO_CORE_PATH)

core_srcs = rebase_sources('build/core', ARDUINO_CORE_PATH)
core_objs = env.Object(core_srcs)

env.VariantDir('build/variant', ARDUINO_VARIANT_PATH)

vari_srcs = rebase_sources('build/variant', ARDUINO_VARIANT_PATH)
vari_objs = env.Object(vari_srcs)

core =  env.StaticLibrary('core', source = [core_objs, vari_objs])

env.VariantDir('build/project', 'project')

proj_srcs = rebase_sources('build/project', 'project')
proj_objs = env.Object(proj_srcs)

proj =  env.StaticLibrary('proj', source = [proj_objs])

env.VariantDir('build', '.')

out_elf = env.Program(target='out.elf', source = [])

env.Depends(out_elf, [core, proj])

out_bin = env.Command('out.bin', out_elf, '%s -O binary $SOURCES $TARGET' %
                      arduino_gcc_bin('objcopy'))

bossac_cmd = '%s --port=%s -U false -e -w -v -b $SOURCES -R' % (
    os.path.join(ARDUINO_TOOLS_PATH, 'bossac'),
    os.path.basename(ARDUINO_PORT)
)

upload = env.Alias('upload', out_bin, [touch_port_1200, touch_port_9600, bossac_cmd])

env.AlwaysBuild(upload)

env.Clean('all', 'build')
