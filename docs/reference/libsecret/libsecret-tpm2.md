Title: Extending file backend to use a TPM
Slug: libsecret-tpm2

# Extending file backend to use a TPM

##  Introduction

The current implementation of file backend uses an encryption key derived from
the user's login password. Security wise this not an ideal situation. Because,
the entire security of the file backend relies on the user's login password
(single point of failure). This situation can be improved if the keys are
protected/generated by hardware. A Trusted Platform Module (TPM) is a such
hardware security module found in modern computer systems.

The new `EGG_TPM2` API based on the "TSS Enhanced System API (ESAPI)"

```c
EggTpm2Context *egg_tpm2_initialize               (GError **);
void            egg_tpm2_finalize                 (EggTpm2Context *);
GBytes         *egg_tpm2_generate_master_password (EggTpm2Context *,
                                                   GError **);
GBytes         *egg_tpm2_decrypt_master_password  (EggTpm2Context *,
                                                   GBytes *,
                                                   GError **);
```

## Build and test libsecret with TPM2 support

In order to try out the `TPM2 support` use the `-Dtpm2=true` build option/flag
during the `meson _build` process.

You can alter the default build and install process as the following:

```sh
meson _build -Dtpm2=true
ninja -C _build
ninja -C _build install
```

For testing the TPM2 support you need a physical TPM or a TPM emulator. The
following sections demonstrate how to setup
[swtpm](https://github.com/stefanberger/swtpm) emulator and testing out the TPM2
support. If you have access to a TPM you can ignore the emulator section.

swtpm emulator setup:

```sh
dnf install swtpm swtpm-tools tpm2-abrmd tpm2-tss-devel
eval `dbus-launch --sh-syntax`
export XDG_CONFIG_HOME=$HOME/.config/swtpm
/usr/share/swtpm/swtpm-create-user-config-files --root
mkdir -p ${XDG_CONFIG_HOME}/mytpm1
swtpm_setup --tpm2 --tpmstate $XDG_CONFIG_HOME/mytpm1 --createek --allow-signing --decryption --create-ek-cert --create-platform-cert --lock-nvram --overwrite --display
swtpm socket --tpm2 --tpmstate dir=$XDG_CONFIG_HOME/mytpm1 --flags startup-clear --ctrl type=tcp,port=2322 --server type=tcp,port=2321 --daemon
tpm2-abrmd --logger=stdout --tcti=swtpm: --session --allow-root --flush-all &amp;
export TCTI=tabrmd:bus_type=session
```

Test TPM2 support:

```sh
cd libsecret
meson _build -Dtpm2=true
ninja -C _build
export SECRET_BACKEND=file
export SECRET_FILE_TEST_PATH=$PWD/keyring
./_build/tool/secret-tool store --label=foo bar baz
ls # keyring
```