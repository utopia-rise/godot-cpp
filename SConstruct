#!python

import os, subprocess, platform, sys


def add_sources(sources, dir, extension):
  for f in os.listdir(dir):
      if f.endswith('.' + extension):
          sources.append(dir + '/' + f)

def sys_exec(args):
    proc = subprocess.Popen(args, stdout=subprocess.PIPE)
    (out, err) = proc.communicate()
    return out.rstrip("\r\n").lstrip()

march_android_switcher = {
    "arm": "armv7-a",
    "arm64": "armv8-a",
    "x86": "i686",
    "x86_64": "x86-64"
}

# Try to detect the host platform automatically
# This is used if no `platform` argument is passed
if sys.platform.startswith('linux'):
    host_platform = 'linux'
elif sys.platform == 'darwin':
    host_platform = 'osx'
elif sys.platform == 'win32':
    host_platform = 'windows'
else:
    raise ValueError('Could not detect platform automatically, please specify with platform=<platform>')

opts = Variables([], ARGUMENTS)

opts.Add(EnumVariable('platform', 'Target platform', host_platform,
                    allowed_values=('linux', 'osx', 'windows', 'android', 'ios'),
                    ignorecase=2))
opts.Add(EnumVariable('bits', 'Target platform bits', 'default', ('default', '32', '64')))
opts.Add(BoolVariable('use_llvm', 'Use the LLVM compiler - only effective when targeting Linux', False))
opts.Add(BoolVariable('use_mingw', 'Use the MinGW compiler - only effective on Windows', False))
# Must be the same setting as used for cpp_bindings
opts.Add(EnumVariable('target', 'Compilation target', 'debug',
                    allowed_values=('debug', 'release'),
                    ignorecase=2))
opts.Add(PathVariable('headers_dir', 'Path to the directory containing Godot headers', 'godot_headers', PathVariable.PathIsDir))
opts.Add(BoolVariable('use_custom_api_file', 'Use a custom JSON API file', False))
opts.Add(PathVariable('custom_api_file', 'Path to the custom JSON API file', None, PathVariable.PathIsFile))
opts.Add(BoolVariable('generate_bindings', 'Generate GDNative API bindings', False))

clang_path = ARGUMENTS.get("clang-path", "")

ndk_path = ARGUMENTS.get("ndk-path", "~/Library/android-sdks/ndk-bundle/")
android_api = ARGUMENTS.get("android-api", "21")
android_abi = ARGUMENTS.get("android-abi", "arm")
ndk_toolchain = ARGUMENTS.get("ndk-toolchain", "/tmp/android-" + android_api + "-" + android_abi + "-toolchain")

unknown = opts.UnknownVariables()
if unknown:
    print("Unknown variables:" + unknown.keys())
    Exit(1)

env = Environment()
opts.Update(env)
Help(opts.GenerateHelpText(env))

# This makes sure to keep the session environment variables on Windows
# This way, you can run SCons in a Visual Studio 2017 prompt and it will find all the required tools
if env['platform'] == 'windows':
    if env['bits'] == '64':
        env = Environment(ENV = os.environ, TARGET_ARCH='amd64')
    elif env['bits'] == '32':
        env = Environment(ENV = os.environ, TARGET_ARCH='x86')
    else:
        print("Warning: bits argument not specified, target arch is=" + env['TARGET_ARCH'])
    opts.Update(env)
elif env['platform'] == "ios":
    SDK_MIN_VERSION = "8.0"
    # we could do better to automatically find the right sdk version
    SDK_VERSION = "12.2"
    IOS_PLATFORM_SDK = sys_exec(["xcode-select", "-p"]) + "/Platforms"
    env["CXX"] = sys_exec(["xcrun", "-sdk", "iphoneos", "-find", "clang++"])
elif env['platform'] == "android":
    suffix = "/bin/"
    if android_abi == "arm":
        suffix += "arm-linux-androideabi"
        ranlib = ndk_toolchain + "/arm-linux-androideabi/bin/ranlib"
    elif android_abi == "arm64":
        suffix += "aarch64-linux-android"
        ranlib = ndk_toolchain + "/aarch64-linux-android/bin/ranlib"
    elif android_abi == "x86":
        suffix += "i686-linux-android"
        ranlib = ndk_toolchain + "/i686-linux-android/bin/ranlib"
    elif android_abi == "x86_64":
        suffix += "x86_64-linux-android"
        ranlib = ndk_toolchain + "/x86_64-linux-android/bin/ranlib"
    env["AR"] = ndk_toolchain + suffix + "-ar"
    env["AS"] = ndk_toolchain + suffix + "-as"
    env["CC"] = ndk_toolchain + suffix + "-clang"
    env["CXX"] = ndk_toolchain + suffix + "-clang++"
    env["LD"] = ndk_toolchain + suffix + "-ld"
    env["STRIP"] = ndk_toolchain + suffix + "-strip"
    env["RANLIB"] = ranlib


is64 = False
if (env['TARGET_ARCH'] == 'amd64' or env['TARGET_ARCH'] == 'emt64' or env['TARGET_ARCH'] == 'x86_64'):
    is64 = True
if env['bits'] == 'default':
    env['bits'] = '64' if is64 else '32'

if env['platform'] == 'linux':
    if env['use_llvm']:
        env['CXX'] = '%sclang++' % clang_path

    env.Append(CCFLAGS=['-fPIC', '-g', '-std=c++14', '-Wwrite-strings'])
    env.Append(LINKFLAGS=["-Wl,-R,'$$ORIGIN'"])

    if env['target'] == 'debug':
        env.Append(CCFLAGS=['-Og'])
    elif env['target'] == 'release':
        env.Append(CCFLAGS=['-O3'])

    if env['bits'] == '64':
        env.Append(CCFLAGS=['-m64'])
        env.Append(LINKFLAGS=['-m64'])
    elif env['bits'] == '32':
        env.Append(CCFLAGS=['-m32'])
        env.Append(LINKFLAGS=['-m32'])

elif env['platform'] == 'osx':
    # Use Clang on macOS by default
    env['CXX'] = '%sclang++' % clang_path
    if env['bits'] == '32':
        raise ValueError('Only 64-bit builds are supported for the macOS target.')

    env.Append(CCFLAGS=['-g', '-std=c++14', '-arch', 'x86_64'])
    env.Append(LINKFLAGS=['-arch', 'x86_64', '-framework', 'Cocoa', '-Wl,-undefined,dynamic_lookup'])

    if env['target'] == 'debug':
        env.Append(CCFLAGS=['-Og'])
    elif env['target'] == 'release':
        env.Append(CCFLAGS=['-O3'])

elif env['platform'] == 'ios':
    env.Append(CCFLAGS = ['-g','-O3', '-std=c++11', '-arch', 'arm64', '-arch', 'armv7', '-arch', 'armv7s', '-isysroot', '%s/iPhoneOS.platform/Developer/SDKs/iPhoneOS%s.sdk' % (IOS_PLATFORM_SDK, SDK_VERSION) , '-miphoneos-version-min=%s' % SDK_MIN_VERSION])
    env.Append(LINKFLAGS = ['-arch', 'arm64', '-arch', 'armv7', '-arch', 'armv7s', '-isysroot', '%s/iPhoneOS.platform/Developer/SDKs/iPhoneOS%s.sdk' % (IOS_PLATFORM_SDK, SDK_VERSION) , '-miphoneos-version-min=%s' % SDK_MIN_VERSION, '-Wl,-undefined,dynamic_lookup'])

elif env['platform'] == 'android':
    march = march_android_switcher.get(android_abi, "Invalid android architecture")
    env.Append(CCFLAGS = ['-fPIE', '-fPIC', '-mfpu=neon', '-march=' + march])
    env.Append(LDFLAGS = ['-pie', '-Wl'])

elif env['platform'] == 'windows':
    if host_platform == 'windows' and not env['use_mingw']:
        # MSVC
        env.Append(LINKFLAGS=['/WX'])
        if env['target'] == 'debug':
            env.Append(CCFLAGS=['/EHsc', '/D_DEBUG', '/MDd'])
        elif env['target'] == 'release':
            env.Append(CCFLAGS=['/O2', '/EHsc', '/DNDEBUG', '/MD'])
    else:
        # MinGW
        if env['bits'] == '64':
            env['CXX'] = 'x86_64-w64-mingw32-g++'
        elif env['bits'] == '32':
            env['CXX'] = 'i686-w64-mingw32-g++'

        env.Append(CCFLAGS=['-g', '-O3', '-std=c++14', '-Wwrite-strings'])
        env.Append(LINKFLAGS=['--static', '-Wl,--no-undefined', '-static-libgcc', '-static-libstdc++'])


env.Append(CPPPATH=['.', env['headers_dir'], 'include', 'include/gen', 'include/core'])

# Generate bindings?
json_api_file = ''

if env['use_custom_api_file']:
    json_api_file = env['custom_api_file']
else:
    json_api_file = os.path.join(os.getcwd(), 'godot_headers', 'api.json')

if env['generate_bindings']:
    # Actually create the bindings here

    import binding_generator

    binding_generator.generate_bindings(json_api_file)

# source to compile
sources = []
add_sources(sources, 'src/core', 'cpp')
add_sources(sources, 'src/gen', 'cpp')


#TODO : add arch in name for android output
pattern = 'bin/' + 'libgodot-cpp.{}.{}'
if env['platform'] == "android":
    pattern += '.' + march_android_switcher.get(android_abi, "Invalid android architecture") + '.a'
elif env['platform'] == 'ios':
    pattern += '.a'
else:
    pattern += '.{}'

if env['platform'] != 'android':
    library = env.StaticLibrary(
            target=pattern.format(env['platform'], env['target'], env['bits']), source=sources
    )
else:
    library = env.StaticLibrary(target=pattern.format(env['platform'], env['target']), source=sources)
Default(library)
