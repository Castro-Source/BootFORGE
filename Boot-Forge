#!/usr/bin/env python3
# coding: utf-8
import sys
import subprocess
import time
import requests
import os
import re
import threading
import traceback
import socket
import signal
from PyQt5.QtWidgets import *
from PyQt5.QtCore import pyqtSignal, Qt, QThread, QMetaObject, Q_ARG, QObject, QTimer, QThreadPool, QTranslator
from PyQt5.QtGui import QPixmap, QColor, QPainter
import qrcode

APP_STYLE = """
QWidget {
    background:#1B1B1B;
    color:#E0E0E0;
    font-size:13px;
}
QPushButton {
    background:#3A3A3A;
    border:2px solid #555;
    padding:8px;
    border-radius:5px;
    font-weight:bold;
}
QPushButton:hover { background:#505050; }
QPushButton:pressed { background:#2A2A2A; }
QLineEdit, QComboBox {
    background:#2A2A2A;
    border:1px solid #666;
    padding:6px;
    border-radius:4px;
}
QProgressBar {
    border:1px solid #444;
    text-align:center;
    background:#222;
    height:18px;
}
QProgressBar::chunk {
    background:#00FF00;
}
QLabel.warning {
    color:#FF4444;
    font-weight:bold;
    font-size:14px;
}
QTabWidget::pane {
    border: 2px solid #8B0000;
    background:#1B1B1B;
}
QTabBar::tab {
    background:#2A2A2A;
    padding:8px 16px;
    margin-right:2px;
    border-top-left-radius:4px;
    border-top-right-radius:4px;
}
QTabBar::tab:selected {
    background:#505050;
    border-bottom:3px solid #FF4444;
    color:white;
}
#BootForge {
    border: 3px solid #8B0000;
    border-radius: 8px;
}
"""

# ===================== CLASSE CONSOLE =====================
class TabConsole(QWidget):
    def __init__(self, hide_progress=False):
        super().__init__()
        layout = QVBoxLayout(self)
        self.progress = QProgressBar()
        self.log = QTextEdit()
        self.log.setReadOnly(True)
        self.log.setStyleSheet("font-family: monospace; background:#111;")
        if hide_progress:
            self.progress.hide()
        layout.addWidget(self.progress)
        layout.addWidget(self.log)

    def write(self, text):
        QMetaObject.invokeMethod(self.log, "append", Qt.QueuedConnection, Q_ARG(str, str(text).strip()))

    def start_indeterminate(self):
        QMetaObject.invokeMethod(self.progress, "setRange", Qt.QueuedConnection, Q_ARG(int, 0), Q_ARG(int, 0))

    def stop_indeterminate(self):
        QMetaObject.invokeMethod(self.progress, "setRange", Qt.QueuedConnection, Q_ARG(int, 0), Q_ARG(int, 100))
        QMetaObject.invokeMethod(self.progress, "setValue", Qt.QueuedConnection, Q_ARG(int, 100))

# ===================== AUXILIARES =====================
def is_dangerous_device(dev):
    return any(x in dev.lower() for x in ["/dev/sda", "/dev/nvme0n1"])

def is_any_mounted(dev):
    if not dev: return False
    try:
        out = subprocess.check_output(["lsblk", "-no", "MOUNTPOINT", dev], text=True, stderr=subprocess.DEVNULL)
        return any(line.strip() for line in out.splitlines() if line.strip())
    except:
        return False

def auto_umount(dev, log_func):
    if is_any_mounted(dev):
        log_func(f"🔄 Desmontando {dev} e partições...")
        try:
            mountpoints = [line.strip() for line in subprocess.check_output(
                ["lsblk", "-no", "MOUNTPOINT", dev], text=True, stderr=subprocess.DEVNULL).splitlines() if line.strip()]
            for mp in reversed(mountpoints):
                subprocess.run(["pkexec", "umount", "-l", mp], check=False)
            subprocess.run(["pkexec", "umount", "-l", dev], check=False)
            log_func("✅ Desmontagem ok.")
        except Exception as e:
            log_func(f"⚠️ Erro desmontagem: {e}")

def get_partition_name(dev):
    if re.search(r"nvme\d+n\d+$", dev): return dev + "p1"
    return dev + "1"

# ===================== WORKER GENÉRICO =====================
class CommandWorker(QObject):
    log_signal = pyqtSignal(str)
    progress_signal = pyqtSignal(bool)
    finished_signal = pyqtSignal(int)

    def __init__(self, cmd, input_text=None):
        super().__init__()
        self.cmd = cmd
        self.input_text = input_text

    def run(self):
        self.progress_signal.emit(True)
        try:
            proc = subprocess.Popen(
                self.cmd,
                stdout=subprocess.PIPE,
                stderr=subprocess.STDOUT,
                text=True,
                stdin=subprocess.PIPE if self.input_text else None,
                bufsize=1
            )
            if self.input_text:
                proc.stdin.write(self.input_text)
                proc.stdin.flush()
                proc.stdin.close()
            for line in iter(proc.stdout.readline, ''):
                if line.strip():
                    self.log_signal.emit(line.strip())
            exit_code = proc.wait()
            self.progress_signal.emit(False)
            self.finished_signal.emit(exit_code)
        except Exception as e:
            self.log_signal.emit(f"ERRO: {str(e)}\n{traceback.format_exc()}")
            self.progress_signal.emit(False)
            self.finished_signal.emit(-1)

# ===================== SINAIS DA BETA =====================
class WorkerSignals(QObject):
    log = pyqtSignal(str)
    progress = pyqtSignal(int)
    busy = pyqtSignal(bool)

def emit_progress(signals, value):
    signals.progress.emit(value)
    signals.log.emit(f"📊 Progresso: {value}%")

def run_cmd(cmd, signals, start=0, end=100, input_text=None):
    signals.log.emit("Executando: " + " ".join(cmd))
    signals.busy.emit(True)
    step_range = end - start
    current = start
    try:
        if input_text:
            proc = subprocess.Popen(cmd, stdin=subprocess.PIPE,
                                    stdout=subprocess.PIPE, stderr=subprocess.STDOUT,
                                    text=True)
            out, _ = proc.communicate(input=input_text+"\n")
            for line in out.splitlines():
                signals.log.emit(line)
        else:
            proc = subprocess.Popen(cmd, stdout=subprocess.PIPE, stderr=subprocess.STDOUT, text=True)
            for line in iter(proc.stdout.readline, ''):
                if not line: break
                signals.log.emit(line.strip())
                if current < end:
                    current += step_range / 100
                    emit_progress(signals, int(current))
            proc.wait()
        if proc.returncode != 0:
            signals.log.emit(f"⚠️ Comando falhou: {' '.join(cmd)}")
    except Exception as e:
        signals.log.emit(f"❌ Erro: {e}")
    finally:
        signals.busy.emit(False)
        emit_progress(signals, end)

def check_dependencies(commands, signals):
    missing = []
    for cmd in commands:
        if subprocess.run(["which", cmd], stdout=subprocess.DEVNULL).returncode != 0:
            missing.append(cmd)
    if missing:
        signals.log.emit(f"❌ Faltando comandos: {', '.join(missing)}")
        return False
    return True

# ===================== VPN WORKER =====================
class VPNWorker(QThread):
    log = pyqtSignal(str)
    status = pyqtSignal(bool)

    def __init__(self, ligar=True):
        super().__init__()
        self.ligar = ligar

    def run(self):
        try:
            if subprocess.run(["which","tor"], stdout=subprocess.DEVNULL).returncode != 0:
                self.log.emit("❌ TOR não instalado. Instale com: sudo apt install tor")
                return
            if self.ligar:
                self.log.emit("🔐 Ligando VPN (TOR)...")
                subprocess.run(["pkexec","sysctl","-w","net.ipv6.conf.all.disable_ipv6=1"])
                subprocess.run(["pkexec","sysctl","-w","net.ipv6.conf.default.disable_ipv6=1"])
                subprocess.run(["pkexec","iptables","-F"])
                subprocess.run(["pkexec","iptables","-t","nat","-F"])
                resolv_backup = "/tmp/resolv.conf.bootforge"
                subprocess.run(f"pkexec cp /etc/resolv.conf {resolv_backup}", shell=True)
                subprocess.run(f"echo 'nameserver 127.0.0.1' | pkexec tee /etc/resolv.conf", shell=True)
                self.tor_proc = subprocess.Popen(["pkexec","tor","--SocksPort","9050","--TransPort","9040","--DNSPort","53"],
                                                 stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)
                time.sleep(12)
                s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
                try:
                    s.connect(("127.0.0.1", 9050))
                    s.close()
                    self.log.emit("Porta 9050 aberta")
                except:
                    self.log.emit("❌ Porta 9050 não abriu - TOR falhou")
                    self.tor_proc.terminate()
                    return
                self.log.emit("🌍 Redirecionando tráfego...")
                subprocess.run(["pkexec","iptables","-t","nat","-A","OUTPUT","-m","owner","!","--uid-owner","0",
                                "-p","tcp","--syn","-j","REDIRECT","--to-ports","9040"])
                subprocess.run(["pkexec","iptables","-t","nat","-A","OUTPUT","-m","owner","!","--uid-owner","0",
                                "-p","udp","--dport","53","-j","REDIRECT","--to-ports","53"])
                self.status.emit(True)
                self.log.emit("🟢 VPN Ativa (TOR)")
            else:
                self.log.emit("🛑 Desligando VPN...")
                subprocess.run(["pkexec","iptables","-F"])
                subprocess.run(["pkexec","iptables","-t","nat","-F"])
                subprocess.run(["pkexec","pkill","tor"])
                if hasattr(self, 'tor_proc'):
                    self.tor_proc.terminate()
                subprocess.run(f"pkexec cp /tmp/resolv.conf.bootforge /etc/resolv.conf", shell=True)
                subprocess.run("pkexec rm /tmp/resolv.conf.bootforge", shell=True)
                subprocess.run(["pkexec","sysctl","-w","net.ipv6.conf.all.disable_ipv6=0"])
                subprocess.run(["pkexec","sysctl","-w","net.ipv6.conf.default.disable_ipv6=0"])
                self.status.emit(False)
                self.log.emit("🔴 VPN Desligada")
        except Exception as e:
            self.log.emit(f"❌ Erro VPN: {e}")

# ===================== CLASSE PRINCIPAL =====================
class BootForge(QWidget):
    def __init__(self):
        super().__init__()
        self.setObjectName("BootForge")
        self.setWindowTitle("BOOT FORGE 1.5 ⚠ OPERAÇÕES DESTRUTIVAS")
        self.resize(940, 700)
        self.vpn_active = False
        self.vpn_worker = None
        self.backup_proc = None
        self.backup_paused = False
        self.translator = QTranslator()
        self.current_language = "pt_BR"

        main = QVBoxLayout(self)
        main.setContentsMargins(12, 12, 12, 12)

        dev_l = QHBoxLayout()
        self.dev_box = QComboBox()
        self.dev_box.setMinimumHeight(32)
        btn_refresh = QPushButton("↻ Atualizar Dispositivos")
        btn_refresh.clicked.connect(self.refresh)
        dev_l.addWidget(QLabel("Dispositivo alvo:"))
        dev_l.addWidget(self.dev_box, 1)
        dev_l.addWidget(btn_refresh)
        main.addLayout(dev_l)

        self.warning_label = QLabel("")
        self.warning_label.setObjectName("warning")
        main.addWidget(self.warning_label)

        self.tabs = QTabWidget()
        self.tabs.setDocumentMode(True)
        main.addWidget(self.tabs, 1)

        self.refresh()
        self.format_tab()
        self.boot_tab()
        self.crypto_tab()
        self.wipe_tab()
        self.backup_tab()
        self.hdrescue_tab()
        self.vpn_tab()
        self.config_tab()          # ← Aba Configurações adicionada aqui
        self.donation_tab()

        self.setAttribute(Qt.WA_DeleteOnClose)

    def closeEvent(self, event):
        print("Fechando app - parando threads...")
        if hasattr(self, 'vpn_worker') and self.vpn_worker and self.vpn_worker.isRunning():
            self.vpn_worker.quit()
            if not self.vpn_worker.wait(8000):
                print("VPN worker não terminou - forçando kill")
        if self.backup_proc and self.backup_proc.poll() is None:
            self.backup_proc.terminate()
            try:
                self.backup_proc.wait(timeout=5)
            except:
                self.backup_proc.kill()
        QThreadPool.globalInstance().waitForDone(5000)
        event.accept()
        print("App fechado com sucesso")

    def confirm_destructive(self, action, dev):
        msg = QMessageBox(self)
        msg.setIcon(QMessageBox.Critical)
        msg.setWindowTitle("🚨 CONFIRMAÇÃO OBRIGATÓRIA")
        text = f"Você vai {action.upper()} o dispositivo:\n\n{dev}\n\nTODOS OS DADOS SERÃO PERDIDOS!"
        if is_dangerous_device(dev):
            text += "\n\n⚠️⚠️⚠️ DISCO DO SISTEMA DETECTADO!\nRISCO DE TORNAR O PC INOPERANTE!"
        msg.setText(text)
        msg.setInformativeText("Digite DESTRUIR para continuar")
        line = QLineEdit()
        line.setPlaceholderText("DESTRUIR")
        msg.layout().addWidget(line, 4, 0, 1, 2)
        msg.setStandardButtons(QMessageBox.Yes | QMessageBox.No)
        return msg.exec_() == QMessageBox.Yes and line.text().strip().upper() == "DESTRUIR"

    def refresh(self):
        self.dev_box.clear()
        try:
            out = subprocess.check_output(["lsblk", "-dpno", "NAME,SIZE,MODEL"], text=True)
            for line in out.splitlines():
                if line.strip():
                    dev = line.split()[0]
                    display = f"🔴 {line} ⚠️ DISCO DO SISTEMA" if is_dangerous_device(dev) else line
                    self.dev_box.addItem(display)
        except:
            self.dev_box.addItem("/dev/sda (erro ao listar)")

    def warn_if_mounted(self, dev, console):
        r = QMessageBox.warning(self, "Ainda montado",
                                f"{dev} ainda está montado.\nDesmontar agora?",
                                QMessageBox.Yes | QMessageBox.No)
        if r == QMessageBox.Yes:
            auto_umount(dev, console.write)
            return not is_any_mounted(dev)
        return False

    def run_long_command(self, cmd, console, success_msg="", input_text=None):
        thread = QThread(self)
        worker = CommandWorker(cmd, input_text)
        worker.moveToThread(thread)
        thread.started.connect(worker.run)
        worker.log_signal.connect(console.write)
        worker.progress_signal.connect(lambda s: console.start_indeterminate() if s else console.stop_indeterminate())
        worker.finished_signal.connect(lambda code: self.on_finish(code, console, success_msg))
        thread.start()

    def on_finish(self, exit_code, console, success_msg):
        console.stop_indeterminate()
        if exit_code == 0:
            console.write(success_msg or "✅ Concluído!")
        else:
            console.write(f"❌ Falha (código {exit_code})")

    # ===================== CONFIGURAÇÕES =====================
    def config_tab(self):
        tab = QWidget()
        l = QVBoxLayout(tab)
        l.addWidget(QLabel("Configurações do Boot Forge"))

        self.btn_update = QPushButton("Verificar Atualização via GitHub")
        self.btn_update.setStyleSheet("background:#4CAF50; color:white;")
        self.btn_update.clicked.connect(self.check_github_update)
        l.addWidget(self.btn_update)

        l.addWidget(QLabel("Selecionar Idioma:"))
        self.lang_combo = QComboBox()
        self.lang_combo.addItems([
            "Português Brasil",
            "Inglês",
            "Espanhol",
            "Chinês",
            "Japonês",
            "Russo"
        ])
        self.lang_combo.currentIndexChanged.connect(self.change_language)
        l.addWidget(self.lang_combo)

        self.config_console = TabConsole(hide_progress=True)
        l.addWidget(self.config_console)
        l.addStretch()
        self.tabs.addTab(tab, "⚙️ Configurações")

        # Mostra versão atual ao abrir a aba
        self.config_console.write(f"<font color='white'>Versão atual instalada: 1.5</font>")

    def check_github_update(self):
        self.config_console.write("Verificando atualização no GitHub...")
        self.btn_update.setEnabled(False)

        try:
            url = "https://api.github.com/repos/Castro-Source/BootFORGE/releases/latest"
            headers = {'User-Agent': 'BootForge-Checker/1.5'}
            response = requests.get(url, headers=headers, timeout=10)
            response.raise_for_status()
            data = response.json()

            latest_tag = data.get('tag_name', None)
            if not latest_tag:
                self.config_console.write("<font color='orange'>Nenhuma release encontrada no GitHub ainda.</font>")
                self.btn_update.setEnabled(True)
                return

            current_version = "1.5"  # ATUALIZE ESTA LINHA QUANDO LANÇAR NOVA VERSÃO
            latest_version = latest_tag.lstrip('vV')

            if latest_version == current_version:
                self.config_console.write("<font color='lime'>BOOTFORGE JÁ ATUALIZADO ✓</font>")
            else:
                self.config_console.write(f"<font color='yellow'>Nova versão disponível: {latest_version}</font>")
                self.config_console.write(f"<font color='yellow'>Baixe aqui: {data.get('html_url', 'link não encontrado')}</font>")

        except requests.exceptions.HTTPError as e:
            if e.response.status_code == 404:
                self.config_console.write("<font color='orange'>Nenhuma release publicada ainda no GitHub.</font>")
            else:
                self.config_console.write(f"<font color='red'>Erro ao acessar GitHub: {e}</font>")
        except requests.exceptions.RequestException as e:
            self.config_console.write(f"<font color='red'>Erro de conexão: {str(e)}</font>")
        except Exception as e:
            self.config_console.write(f"<font color='red'>Erro inesperado: {str(e)}</font>")

        self.btn_update.setEnabled(True)

    def change_language(self):
        lang_map = {
            0: ("pt_BR", "Português Brasil"),
            1: ("en", "Inglês"),
            2: ("es", "Espanhol"),
            3: ("zh_CN", "Chinês"),
            4: ("ja", "Japonês"),
            5: ("ru", "Russo")
        }

        index = self.lang_combo.currentIndex()
        lang_code, lang_name = lang_map.get(index, ("pt_BR", "Português Brasil"))

        translator = QTranslator()
        qm_file = f"translations/bootforge_{lang_code}.qm"

        if translator.load(qm_file):
            QApplication.instance().installTranslator(translator)
            self.config_console.write(f"<font color='lime'>Idioma alterado para {lang_name} ✓</font>")
            self.config_console.write("Reinicie o programa para aplicar completamente.")
        else:
            self.config_console.write(f"<font color='orange'>Arquivo de tradução não encontrado: {qm_file}</font>")
            self.config_console.write("Crie o arquivo com Qt Linguist e coloque na pasta translations/")

    # ===================== FORMATAÇÃO =====================
    def format_tab(self):
        tab = QWidget()
        l = QVBoxLayout(tab)
        self.fs = QComboBox()
        self.fs.addItems(["NTFS", "FAT32", "EXT4", "exFAT"])
        self.label = QLineEdit("BOOTFORGE")
        self.quick = QCheckBox("Formatação RÁPIDA (Quick Format)")
        btn = QPushButton("🖴 FORMATAR DISPOSITIVO")
        btn.setStyleSheet("background:#FF4444; color:white;")
        self.format_console = TabConsole()
        self.format_console.write("Clique em FORMATAR para iniciar.")
        l.addWidget(QLabel("Sistema de arquivos:"))
        l.addWidget(self.fs)
        l.addWidget(QLabel("Nome do volume:"))
        l.addWidget(self.label)
        l.addWidget(self.quick)
        l.addWidget(btn)
        l.addWidget(self.format_console)
        self.tabs.addTab(tab, "🖴 Formatação")

        def start():
            dev_raw = self.dev_box.currentText()
            dev = dev_raw.split()[0] if dev_raw else ""
            if not dev or not self.confirm_destructive("FORMATAR", dev):
                return
            auto_umount(dev, self.format_console.write)
            if is_any_mounted(dev):
                if not self.warn_if_mounted(dev, self.format_console):
                    return
            signals = WorkerSignals()
            signals.log.connect(self.format_console.write)
            signals.progress.connect(self.format_console.progress.setValue)
            threading.Thread(target=lambda: self.format_worker(dev, signals), daemon=True).start()
        btn.clicked.connect(start)

    def format_worker(self, dev, signals):
        if not check_dependencies(["mkfs.ext4","mkfs.ntfs","mkfs.vfat","mkfs.exfat","wipefs","parted"], signals): return
        run_cmd(["pkexec","wipefs","-a",dev], signals, 5, 15)
        run_cmd(["pkexec","parted","-s",dev,"mklabel","gpt"], signals, 15, 25)
        run_cmd(["pkexec","parted","-s",dev,"mkpart","primary","0%","100%"], signals, 25, 35)
        part = get_partition_name(dev)
        label = self.label.text() or "BOOTFORGE"
        fs = self.fs.currentText()
        if fs == "NTFS":
            cmd = ["pkexec","mkfs.ntfs","-F","-L",label,part]
            if self.quick.isChecked(): cmd = ["pkexec","mkfs.ntfs","-F","-f","-L",label,part]
        elif fs == "FAT32":
            cmd = ["pkexec","mkfs.vfat","-F","32","-n",label[:11],part]
        elif fs == "exFAT":
            cmd = ["pkexec","mkfs.exfat","-n",label,part]
        else:
            cmd = ["pkexec","mkfs.ext4","-F","-L",label,part]
        run_cmd(cmd, signals, 40, 100)

    # ===================== GRAVAR ISO =====================
    def boot_tab(self):
        tab = QWidget()
        l = QVBoxLayout(tab)
        self.iso = QLineEdit()
        self.iso.setPlaceholderText("Caminho do arquivo .iso")
        btn_pick = QPushButton("📂 Selecionar ISO")
        btn_write = QPushButton("💿 GRAVAR ISO BOOTÁVEL")
        btn_write.setStyleSheet("background:#FF4444; color:white;")
        self.boot_console = TabConsole()
        self.boot_console.write("Clique em GRAVAR ISO para iniciar.")
        l.addWidget(self.iso)
        l.addWidget(btn_pick)
        l.addWidget(btn_write)
        l.addWidget(self.boot_console)
        self.tabs.addTab(tab, "💿 Gravar ISO")

        btn_pick.clicked.connect(lambda: self.iso.setText(QFileDialog.getOpenFileName(self, "Selecionar ISO", "", "ISO (*.iso)")[0]))

        def start():
            dev_raw = self.dev_box.currentText()
            dev = dev_raw.split()[0] if dev_raw else ""
            iso = self.iso.text().strip()
            if not dev or not iso or not self.confirm_destructive("GRAVAR ISO", dev):
                return
            auto_umount(dev, self.boot_console.write)
            signals = WorkerSignals()
            signals.log.connect(self.boot_console.write)
            signals.progress.connect(self.boot_console.progress.setValue)
            cmd = ["pkexec","dd",f"if={iso}",f"of={dev}","bs=4M","status=progress","conv=fsync"]
            threading.Thread(target=lambda: run_cmd(cmd, signals, 10, 100), daemon=True).start()
        btn_write.clicked.connect(start)

    # ===================== CRIPTOGRAFIA =====================
    def crypto_tab(self):
        tab = QWidget()
        l = QVBoxLayout(tab)
        btn = QPushButton("🔐 CRIPTOGRAFAR COM LUKS2")
        btn.setStyleSheet("background:#FF4444; color:white;")
        self.crypto_console = TabConsole()
        self.crypto_console.write("Clique para iniciar.")
        l.addWidget(btn)
        l.addWidget(self.crypto_console)
        self.tabs.addTab(tab, "🔐 Criptografia")

        def start():
            dev_raw = self.dev_box.currentText()
            dev = dev_raw.split()[0] if dev_raw else ""
            if not dev or not self.confirm_destructive("CRIPTOGRAFAR", dev):
                return
            auto_umount(dev, self.crypto_console.write)
            p1, ok1 = QInputDialog.getText(self, "Senha LUKS", "Digite a senha:", QLineEdit.Password)
            if not ok1 or not p1: return
            p2, ok2 = QInputDialog.getText(self, "Confirmação", "Confirme:", QLineEdit.Password)
            if not ok2 or p1 != p2:
                self.crypto_console.write("❌ Senhas diferentes!")
                return
            signals = WorkerSignals()
            signals.log.connect(self.crypto_console.write)
            signals.progress.connect(self.crypto_console.progress.setValue)
            threading.Thread(target=lambda: self.encrypt_worker(dev, p1, signals), daemon=True).start()
        btn.clicked.connect(start)

    def encrypt_worker(self, dev, password, signals):
        if not check_dependencies(["cryptsetup","mkfs.ext4"], signals): return
        part = get_partition_name(dev)
        run_cmd(["pkexec","cryptsetup","luksFormat",part,"--key-file=-"], signals, 40, 70, password)
        run_cmd(["pkexec","cryptsetup","open",part,"bootforge_luks","--key-file=-"], signals, 70, 85, password)
        run_cmd(["pkexec","mkfs.ext4","/dev/mapper/bootforge_luks"], signals, 85, 95)
        run_cmd(["pkexec","cryptsetup","close","bootforge_luks"], signals, 95, 100)
        signals.log.emit("✅ Criptografado")

    # ===================== WIPE =====================
    def wipe_tab(self):
        tab = QWidget()
        l = QVBoxLayout(tab)
        self.wipe_method = QComboBox()
        self.wipe_method.addItems(["SSD (blkdiscard)", "HDD 1 passagem", "HDD 3 passagens"])
        self.confirm_text = QLineEdit()
        self.confirm_text.setPlaceholderText("Digite APAGAR")
        btn = QPushButton("💀 DESTRUIR")
        btn.setStyleSheet("background:#8B0000; color:white;")
        self.wipe_console = TabConsole()
        self.wipe_console.write("Clique em DESTRUIR.")
        l.addWidget(QLabel("Método:"))
        l.addWidget(self.wipe_method)
        l.addWidget(QLabel("Confirmação:"))
        l.addWidget(self.confirm_text)
        l.addWidget(btn)
        l.addWidget(self.wipe_console)
        self.tabs.addTab(tab, "💀 Destruir Disco")

        def start():
            dev_raw = self.dev_box.currentText()
            dev = dev_raw.split()[0] if dev_raw else ""
            if not dev or not self.confirm_destructive("DESTRUIR", dev):
                return
            if self.confirm_text.text().strip().upper() != "APAGAR":
                self.wipe_console.write("❌ Digite APAGAR")
                return
            auto_umount(dev, self.wipe_console.write)
            signals = WorkerSignals()
            signals.log.connect(self.wipe_console.write)
            signals.progress.connect(self.wipe_console.progress.setValue)
            threading.Thread(target=lambda: self.wipe_worker(dev, signals), daemon=True).start()
        btn.clicked.connect(start)

    def wipe_worker(self, dev, signals):
        if not check_dependencies(["shred","blkdiscard"], signals): return
        method = self.wipe_method.currentText()
        if "SSD" in method:
            run_cmd(["pkexec","blkdiscard",dev], signals, 10, 100)
        elif "1 passagem" in method:
            run_cmd(["pkexec","shred","-v","-n","1","-z",dev], signals, 10, 100)
        else:
            run_cmd(["pkexec","shred","-v","-n","3","-z",dev], signals, 10, 100)
        signals.log.emit("💀 Destruído")

    # ===================== BACKUP =====================
    def backup_tab(self):
        tab = QWidget()
        l = QVBoxLayout(tab)
        self.src = QLineEdit()
        self.dst = QLineEdit()
        btn_src = QPushButton("📂 Origem...")
        btn_dst = QPushButton("📂 Destino...")
        btn_start = QPushButton("▶ INICIAR BACKUP")
        btn_start.setStyleSheet("background:#00AA00; color:white;")
        btn_pause = QPushButton("⏸ Pausar")
        btn_stop = QPushButton("⏹ Cancelar")
        self.backup_console = TabConsole()
        self.backup_console.write("Selecione origem e destino.")
        l.addWidget(QLabel("Origem:"))
        l.addWidget(self.src)
        l.addWidget(btn_src)
        l.addWidget(QLabel("Destino:"))
        l.addWidget(self.dst)
        l.addWidget(btn_dst)
        btn_layout = QHBoxLayout()
        btn_layout.addWidget(btn_start, 2)
        btn_layout.addWidget(btn_pause)
        btn_layout.addWidget(btn_stop)
        l.addLayout(btn_layout)
        l.addWidget(self.backup_console)
        self.tabs.addTab(tab, "📦 Backup")

        btn_src.clicked.connect(lambda: self.src.setText(QFileDialog.getExistingDirectory(self, "Origem")))
        btn_dst.clicked.connect(lambda: self.dst.setText(QFileDialog.getExistingDirectory(self, "Destino")))

        def start():
            src = self.src.text().rstrip("/") + "/"
            dst = self.dst.text().rstrip("/")
            if not src or not dst:
                self.backup_console.write("❌ Selecione origem e destino")
                return
            self.backup_console.start_indeterminate()
            cmd = ["rsync", "-aAXvH", "--info=progress2", src, dst]
            thread = QThread(self)
            worker = CommandWorker(cmd)
            worker.moveToThread(thread)
            thread.started.connect(worker.run)
            worker.log_signal.connect(self.backup_console.write)
            worker.progress_signal.connect(lambda s: self.backup_console.start_indeterminate() if s else self.backup_console.stop_indeterminate())
            worker.finished_signal.connect(lambda code: self.on_finish(code, self.backup_console, "✅ Backup concluído!"))
            thread.start()
            self.backup_proc = worker
        btn_start.clicked.connect(start)

        def pause():
            if self.backup_paused:
                subprocess.run(["pkill", "-CONT", "rsync"])
                btn_pause.setText("⏸ Pausar")
                self.backup_paused = False
                self.backup_console.write("▶ Backup continuado")
            else:
                subprocess.run(["pkill", "-STOP", "rsync"])
                btn_pause.setText("▶ Continuar")
                self.backup_paused = True
                self.backup_console.write("⏸ Backup pausado")
        def cancel():
            subprocess.run(["pkill", "rsync"])
            self.backup_console.write("⏹ Backup cancelado")
            self.backup_console.stop_indeterminate()
        btn_pause.clicked.connect(pause)
        btn_stop.clicked.connect(cancel)

    # ===================== HD RESCUE =====================
    def hdrescue_tab(self):
        tab = QWidget()
        l = QVBoxLayout(tab)
        self.rec_src = QLineEdit()
        self.rec_dst = QLineEdit()
        btn_src = QPushButton("📂 Origem (disco ou imagem)")
        btn_dst = QPushButton("📂 Destino (pasta)")
        btn_photorec = QPushButton("🔍 Photorec (Deep File Scan)")
        btn_photorec.setStyleSheet("background:#FFAA00; color:black;")
        btn_testdisk = QPushButton("🔍 Testdisk (Partições + Deep)")
        btn_testdisk.setStyleSheet("background:#FF8800; color:black;")
        self.rescue_console = TabConsole()
        self.rescue_console.write("Selecione origem e destino.")
        l.addWidget(QLabel("Origem:"))
        l.addWidget(self.rec_src)
        l.addWidget(btn_src)
        l.addWidget(QLabel("Destino:"))
        l.addWidget(self.rec_dst)
        l.addWidget(btn_dst)
        btn_layout = QHBoxLayout()
        btn_layout.addWidget(btn_photorec)
        btn_layout.addWidget(btn_testdisk)
        l.addLayout(btn_layout)
        l.addWidget(self.rescue_console)
        self.tabs.addTab(tab, "🔍 HD Rescue")

        btn_src.clicked.connect(lambda: self.rec_src.setText(QFileDialog.getOpenFileName(self, "Origem", "", "Todos (*)")[0]))
        btn_dst.clicked.connect(lambda: self.rec_dst.setText(QFileDialog.getExistingDirectory(self, "Destino")))

        def start_photorec():
            src = self.rec_src.text().strip()
            dst = self.rec_dst.text().strip()
            if not src or not dst: return
            self.rescue_console.start_indeterminate()
            cmd = ["pkexec", "photorec", "/log", "/d", dst, "/cmd", src,
                   "partition_none,options,mode_other,fileopt,everything,enable,search"]
            self.run_long_command(cmd, self.rescue_console, "✅ Photorec concluído")
        btn_photorec.clicked.connect(start_photorec)

        def start_testdisk():
            src = self.rec_src.text().strip()
            if not src: return
            self.rescue_console.start_indeterminate()
            cmd = ["pkexec", "testdisk", src, "/log", "/cmd", "analyse,quick,search,list"]
            self.run_long_command(cmd, self.rescue_console, "✅ Testdisk concluído (verifique log)")
        btn_testdisk.clicked.connect(start_testdisk)

    # ===================== VPN =====================
    def vpn_tab(self):
        tab = QWidget()
        l = QVBoxLayout(tab)
        self.vpn_indicator = QLabel()
        self.set_indicator(False)
        self.status_label = QLabel("VPN: DESLIGADA")
        self.status_label.setAlignment(Qt.AlignCenter)
        self.ip_label = QLabel("IP: —")
        self.country_label = QLabel("País: —")
        self.flag_label = QLabel()
        self.flag_label.setFixedSize(52, 32)
        self.flag_label.setScaledContents(True)
        btn_on = QPushButton("🌐 LIGAR VPN (TOR)")
        btn_off = QPushButton("🛑 DESLIGAR VPN")
        btn_check = QPushButton("🔍 Verificar TOR")
        self.vpn_console = TabConsole(hide_progress=True)
        l.addWidget(self.vpn_indicator, alignment=Qt.AlignCenter)
        l.addWidget(self.status_label)
        ip_l = QHBoxLayout()
        ip_l.addWidget(self.ip_label)
        ip_l.addWidget(self.flag_label)
        ip_l.addWidget(self.country_label)
        l.addLayout(ip_l)
        l.addWidget(btn_on)
        l.addWidget(btn_off)
        l.addWidget(btn_check)
        l.addWidget(self.vpn_console)
        self.tabs.addTab(tab, "🛡️ VPN")

        btn_on.clicked.connect(lambda: self.start_vpn(True))
        btn_off.clicked.connect(lambda: self.start_vpn(False))
        btn_check.clicked.connect(self.update_ip_loop)

    def set_indicator(self, on):
        pix = QPixmap(32, 32)
        pix.fill(Qt.transparent)
        p = QPainter(pix)
        p.setBrush(QColor("#00FF00" if on else "#FF0000"))
        p.setPen(Qt.NoPen)
        p.drawEllipse(6, 6, 20, 20)
        p.end()
        self.vpn_indicator.setPixmap(pix)

    def start_vpn(self, ligar):
        self.vpn_active = ligar
        self.vpn_worker = VPNWorker(ligar)
        self.vpn_worker.log.connect(self.vpn_console.write)
        self.vpn_worker.status.connect(self.update_vpn_status)
        self.vpn_worker.start()
        if ligar:
            QTimer.singleShot(15000, self.update_ip_loop)

    def update_vpn_status(self, ativa):
        self.set_indicator(ativa)
        self.status_label.setText("🟢 VPN ATIVA (TOR)" if ativa else "🔴 VPN DESLIGADA")

    def update_ip_loop(self):
        proxies = {"http": "socks5h://127.0.0.1:9050", "https": "socks5h://127.0.0.1:9050"}
        try:
            r = requests.get("https://ipapi.co/json/", proxies=proxies, timeout=12)
            data = r.json()
            QMetaObject.invokeMethod(self.ip_label, "setText", Qt.QueuedConnection, Q_ARG(str, f"IP: {data.get('ip', '—')}"))
            QMetaObject.invokeMethod(self.country_label, "setText", Qt.QueuedConnection, Q_ARG(str, f"País: {data.get('country_name', '—')}"))
            code = data.get("country_code", "").lower()
            if code:
                img_data = requests.get(f"https://flagcdn.com/w40/{code}.png", proxies=proxies, timeout=8).content
                pix = QPixmap()
                pix.loadFromData(img_data)
                QMetaObject.invokeMethod(self.flag_label, "setPixmap", Qt.QueuedConnection, Q_ARG(QPixmap, pix))
        except Exception as e:
            self.vpn_console.write(f"Falha IP: {e}")

    # ===================== DOAÇÃO =====================
    def donation_tab(self):
        tab = QWidget()
        l = QVBoxLayout(tab)
        l.setSpacing(15)
        lbl_thanks = QLabel("""<h1 style="color:#FFAA00;">Agradecimentos à comunidade Linux</h1>
<h2>Desenvolvido por Andre Castro</h2>
<h3><b>— Fevereiro de 2026 —</b></h3>""")
        lbl_thanks.setAlignment(Qt.AlignCenter)
        l.addWidget(lbl_thanks)
        lbl_doe = QLabel("Doe qualquer valor para apoiar o projeto")
        lbl_doe.setAlignment(Qt.AlignCenter)
        lbl_doe.setStyleSheet("font-size:16px; font-weight:bold;")
        l.addWidget(lbl_doe)
        btc = "bc1qn5d5epsvgruy4txayrz82nxwrrmekjfc593vgx"
        lbl_btc = QLabel(btc)
        lbl_btc.setAlignment(Qt.AlignCenter)
        lbl_btc.setStyleSheet("font-size:18px; font-weight:bold; color:#F7931A; background:#222; padding:8px; border-radius:6px;")
        l.addWidget(lbl_btc)
        qr_path = os.path.expanduser("~/.cache/bootforge-qr.png")
        os.makedirs(os.path.dirname(qr_path), exist_ok=True)
        qr = qrcode.make(btc)
        qr.save(qr_path)
        pix = QPixmap(qr_path).scaled(280, 280, Qt.KeepAspectRatio)
        img = QLabel()
        img.setPixmap(pix)
        img.setAlignment(Qt.AlignCenter)
        l.addWidget(img)
        self.tabs.addTab(tab, "💰 Doação")

if __name__ == "__main__":
    if os.geteuid() != 0:
        print("AVISO: Rode normalmente - senha gráfica será pedida quando necessário")
    app = QApplication(sys.argv)
    app.setStyleSheet(APP_STYLE)
    w = BootForge()
    w.show()
    sys.exit(app.exec_())
