# Devasc
program Python pendeteksi semua network interface dan disable interface yang statusnya down

#!/usr/bin/env python3
import subprocess
import re
import sys
import os

def run_command(cmd):
    """Menjalankan command dan mengembalikan outputnya"""
    try:
        result = subprocess.run(cmd, shell=True, capture_output=True, text=True, check=True)
        return result.stdout
    except subprocess.CalledProcessError as e:
        print(f"Error executing command: {cmd}")
        print(f"Error message: {e.stderr}")
        return None

def get_network_interfaces():
    """Mendapatkan daftar semua network interface dan statusnya"""
    interfaces = {}
    
    # Menggunakan ip link show untuk mendapatkan informasi interface
    output = run_command("ip link show")
    if not output:
        return interfaces
    
    # Parsing output untuk mendapatkan nama interface dan status
    lines = output.split('\n')
    current_interface = None
    
    for line in lines:
        # Mencari baris yang berisi nama interface
        interface_match = re.search(r'^\d+:\s+([^:]+):', line)
        if interface_match:
            current_interface = interface_match.group(1)
            # Cek status UP/DOWN
            if 'state UP' in line:
                interfaces[current_interface] = 'UP'
            elif 'state DOWN' in line:
                interfaces[current_interface] = 'DOWN'
            else:
                interfaces[current_interface] = 'UNKNOWN'
    
    return interfaces

def disable_interface(interface_name):
    """Menonaktifkan network interface"""
    print(f"🔄 Menonaktifkan interface: {interface_name}")
    
    # Down interface
    result = run_command(f"sudo ip link set {interface_name} down")
    if result is None:
        print(f"❌ Gagal menonaktifkan {interface_name}")
        return False
    
    print(f"✅ Berhasil menonaktifkan {interface_name}")
    return True

def is_essential_interface(interface_name):
    """Memeriksa apakah interface termasuk essential (tidak boleh dinonaktifkan)"""
    essential_interfaces = ['lo', 'docker0', 'br-', 'veth', 'virbr']
    
    for essential in essential_interfaces:
        if interface_name.startswith(essential):
            return True
    return False

def main():
    """Fungsi utama"""
    print("🔍 Network Interface Manager")
    print("=" * 40)
    
    # Cek apakah running sebagai root
    if os.geteuid() != 0:
        print("❌ Program harus dijalankan dengan sudo/root privileges!")
        print("   Jalankan dengan: sudo python3 script.py")
        sys.exit(1)
    
    # Dapatkan semua interface
    interfaces = get_network_interfaces()
    
    if not interfaces:
        print("❌ Tidak dapat membaca informasi network interface")
        sys.exit(1)
    
    print("\n📊 Daftar Network Interface:")
    print("-" * 40)
    
    interfaces_to_disable = []
    
    for interface, status in interfaces.items():
        status_icon = "🟢" if status == 'UP' else "🔴" if status == 'DOWN' else "⚫"
        essential = " (Essential)" if is_essential_interface(interface) else ""
        print(f"{status_icon} {interface}: {status}{essential}")
        
        # Tambahkan ke list untuk dinonaktifkan jika status DOWN dan bukan essential
        if status == 'DOWN' and not is_essential_interface(interface):
            interfaces_to_disable.append(interface)
    
    print(f"\n📋 Interface yang akan dinonaktifkan: {len(interfaces_to_disable)}")
    
    if not interfaces_to_disable:
        print("✅ Tidak ada interface DOWN yang perlu dinonaktifkan")
        return
    
    # Konfirmasi sebelum menonaktifkan
    print("\n⚠️  PERINGATAN: Tindakan ini akan menonaktifkan interface network!")
    confirm = input("Apakah Anda yakin ingin melanjutkan? (y/N): ")
    
    if confirm.lower() != 'y':
        print("❌ Dibatalkan oleh pengguna")
        return
    
    # Nonaktifkan interface yang down
    print("\n🚀 Memproses penonaktifan interface...")
    success_count = 0
    
    for interface in interfaces_to_disable:
        if disable_interface(interface):
            success_count += 1
    
    print(f"\n📊 Hasil:")
    print(f"✅ Berhasil dinonaktifkan: {success_count} interface")
    print(f"❌ Gagal dinonaktifkan: {len(interfaces_to_disable) - success_count} interface")
    
    # Tampilkan status terbaru
    print(f"\n🔄 Status terbaru:")
    updated_interfaces = get_network_interfaces()
    for interface, status in updated_interfaces.items():
        status_icon = "🟢" if status == 'UP' else "🔴" if status == 'DOWN' else "⚫"
        print(f"{status_icon} {interface}: {status}")

if __name__ == "__main__":
    main()
