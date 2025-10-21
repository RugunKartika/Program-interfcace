import os
import subprocess
from pathlib import Path

SYS_NET = Path("/sys/class/net")

def get_interfaces():
    if not SYS_NET.exists():
        raise RuntimeError(f"{SYS_NET} not found. Script ini hanya untuk Linux dengan /sys/class/net.")
    return [p.name for p in SYS_NET.iterdir() if p.is_dir()]

def read_operstate(iface):
    path = SYS_NET / iface / "operstate"
    try:
        return path.read_text().strip().lower()
    except Exception:
        return "unknown"

def is_loopback(iface):
    # quick check: loopback biasanya named 'lo' or has flag
    return iface == "lo"

def disable_interface(iface):
    # menjalankan ip link set dev <iface> down
    cmd = ["ip", "link", "set", "dev", iface, "down"]
    try:
        subprocess.run(cmd, check=True, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
        return True, None
    except subprocess.CalledProcessError as e:
        return False, e.stderr.decode().strip() or str(e)

def main(dry_run=False):
    interfaces = get_interfaces()
    to_disable = []

    for iface in interfaces:
        if is_loopback(iface):
            continue
        oper = read_operstate(iface)
        # treat 'down' and 'unknown' as candidates; you can extend checks ('dormant', etc.)
        if oper in ("down", "unknown"):
            to_disable.append((iface, oper))

    if not to_disable:
        print("Semua interface operasionalnya UP. Tidak ada yang di-disable.")
        return 0

    print(f"Ditemukan {len(to_disable)} interface dengan operstate down/unknown:")
    for iface, oper in to_disable:
        print(f" - {iface}: operstate={oper}")

    if dry_run:
        print("\nMode dry-run: tidak melakukan perubahan. Gunakan tanpa --dry-run untuk men-disable.")
        return 0

    print("\nMen-disable interface yang terdeteksi...")
    results = []
    for iface, _ in to_disable:
        ok, err = disable_interface(iface)
        results.append((iface, ok, err))

    print("\nRingkasan tindakan:")
    for iface, ok, err in results:
        if ok:
            print(f" - {iface}: DISABLED")
        else:
            print(f" - {iface}: GAGAL -> {err}")

    return 0

if __name__ == "__main__":
    import argparse
    parser = argparse.ArgumentParser(description="Disable interfaces whose operstate is down.")
    parser.add_argument("--dry-run", action="store_true", help="Jalankan tanpa melakukan perubahan (hanya tampilkan).")
    args = parser.parse_args()
    try:
        exit_code = main(dry_run=args.dry_run)
    except Exception as e:
        print(f"Error: {e}")
        exit_code = 2
    raise SystemExit(exit_code)
