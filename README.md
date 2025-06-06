# battery-anime-csvplot
import sys
import os
import random
import math
import pandas as pd
from PyQt5.QtCore import QTimer
from PyQt5.QtGui import QRadialGradient
from PyQt5.QtWidgets import (QApplication, QGraphicsScene, QWidget, QVBoxLayout, QHBoxLayout, 
                             QGraphicsView, QGraphicsObject, QGraphicsItemGroup, QPushButton, 
                             QDesktopWidget, QMainWindow, QLabel, QCheckBox, QFormLayout, 
                             QInputDialog, QLineEdit, QComboBox, QFileDialog, QSplitter)
from PyQt5 import QtCore
from PyQt5.QtCore import Qt, QRectF, QPropertyAnimation, QPointF, QEasingCurve
from PyQt5.QtGui import QPainter, QColor, QPen, QPainterPath, QFont, QPixmap, QBrush
from PyQt5.QtWidgets import QMessageBox
from matplotlib.backends.backend_qt5agg import FigureCanvasQTAgg as FigureCanvas
from matplotlib.figure import Figure
import matplotlib.dates as mdates
from datetime import datetime


class BatteryWidget(QWidget):
    def __init__(self):
        super().__init__()
        self.setMinimumSize(600, 300)
        self.setStyleSheet("background-color: white;")
        
        # Animation parameters
        self.pulse_factor = 0
        self.pulse_direction = 1
        self.angle = 0
        self.is_charging = False
        self.previous_mode = None
        
        # Define electrode regions first
        self.anode_x_range = (80, 250)
        self.cathode_x_range = (260, 480)
        self.vertical_range = (120, 180)
        
        # Initialize ions
        self.ions = []
        for i in range(10):
            x_position = self.cathode_x_range[0] + (i * (self.cathode_x_range[1] - self.cathode_x_range[0]) / 10)
            self.ions.append({
                'x': x_position,
                'y': self.vertical_range[0] + (i % 3) * 20,
                'size': 6,
                'speed': 2.0,
                'phase': random.uniform(0, 2 * math.pi),
                'type': 'Li+'
            })
        
        # Add FePO4 ions
        self.fepo4_ions = []
        cathode_right_boundary = self.cathode_x_range[1] - 25
        for i in range(3):
            self.fepo4_ions.append({
                'x': cathode_right_boundary,
                'y': 110 + i * 30,
                'size': 8,
                'speed': 0,
                'phase': 0,
                'type': 'FePO4'
            })
        
        self.electrons = []
        for _ in range(6):
            self.electrons.append({
                'x': random.randint(80, 480),
                'y': 40,
                'speed': 2.0,
                'size': 3
            })
        
        # Timer setup
        self.timer = QTimer(self)
        self.timer.timeout.connect(self.update_animation)
        self.timer.start(30)
        
        # Cache colors and properties
        self.colors = {
            'container_pen': QColor(100, 100, 100),
            'container_brush': QColor(220, 220, 220),
            'anode': QColor(76, 175, 80, 200),
            'cathode': QColor(33, 150, 243, 200),
            'separator': QColor(150, 150, 150),
            'terminals': QColor(100, 100, 100),
            'ion': QColor(255, 0, 0),
            'ion_text': QColor(255, 255, 255),
            'electron': QColor(255, 235, 59),
            'electron_text': QColor(0, 0, 0),
            'arrow': QColor(255, 105, 180),
            'labels': QColor(0, 0, 0),
            'fepo4': QColor(255, 255, 0),
            'fepo4_text': QColor(0, 0, 0)
        }
        
        self.pens = {
            'container': QPen(self.colors['container_pen'], 2),
            'circuit': QPen(self.colors['container_pen'], 2),
            'ion_text': QPen(self.colors['ion_text'], 1),
            'fepo4_text': QPen(self.colors['fepo4_text'], 1),
            'arrow': QPen(self.colors['arrow'], 2)
        }
        
        self.fonts = {
            'ion': QFont('Arial', 7),
            'labels': QFont('Arial', 10),
            'fepo4': QFont('Arial', 6)
        }

    def set_charging_mode(self, is_charging):
        if self.previous_mode != is_charging:
            self.is_charging = is_charging
            self.previous_mode = is_charging
            
            if is_charging:
                for i, ion in enumerate(self.ions):
                    x_spacing = (self.cathode_x_range[1] - self.cathode_x_range[0]) / len(self.ions)
                    ion['x'] = self.cathode_x_range[0] + (i * x_spacing)
                    ion['y'] = self.vertical_range[0] + (i % 3) * 20
            else:
                for i, ion in enumerate(self.ions):
                    x_spacing = (self.anode_x_range[1] - self.anode_x_range[0]) / len(self.ions)
                    ion['x'] = self.anode_x_range[0] + (i * x_spacing)
                    ion['y'] = self.vertical_range[0] + (i % 3) * 20

    def update_animation(self):
        try:
            self.pulse_factor += 0.05 * self.pulse_direction
            if self.pulse_factor > 1 or self.pulse_factor < 0:
                self.pulse_direction *= -1

            for ion in self.ions:
                if self.is_charging:
                    ion['x'] -= ion['speed']
                    if ion['x'] < self.anode_x_range[0]:
                        ion['x'] = self.cathode_x_range[1] - random.uniform(0, 20)
                        ion['y'] = random.uniform(self.vertical_range[0], self.vertical_range[1])
                else:
                    ion['x'] += ion['speed']
                    if ion['x'] > self.cathode_x_range[1]:
                        ion['x'] = self.anode_x_range[0] + random.uniform(0, 20)
                        ion['y'] = random.uniform(self.vertical_range[0], self.vertical_range[1])
                
                ion['y'] = min(max(
                    ion['y'] + math.sin(self.angle + ion['phase']) * 0.6,
                    self.vertical_range[0]
                ), self.vertical_range[1])

            for electron in self.electrons:
                if self.is_charging:
                    electron['x'] -= electron['speed']
                    if electron['x'] < 50:
                        electron['x'] = 530
                else:
                    electron['x'] += electron['speed']
                    if electron['x'] > 530:
                        electron['x'] = 50

            self.angle += 0.05
            self.update()

        except Exception as e:
            print(f"Animation error: {e}")

    def paintEvent(self, event):
        painter = QPainter(self)
        painter.setRenderHint(QPainter.Antialiasing)

        # Draw battery container (smaller)
        container_path = QPainterPath()
        container_path.addRoundedRect(70, 80, 440, 140, 15, 15)
        painter.setPen(self.pens['container'])
        painter.setBrush(self.colors['container_brush'])
        painter.drawPath(container_path)

        # Draw electrodes and separator
        painter.setBrush(self.colors['anode'])
        painter.drawRect(80, 95, 170, 110)
        
        separator_center = (250 + 260) / 2
        separator_width = 8
        painter.setBrush(self.colors['separator'])
        painter.drawRect(int(separator_center - separator_width/2), 95, separator_width, 110)
        
        painter.setBrush(self.colors['cathode'])
        painter.drawRect(260, 95, 220, 110)

        # Draw terminals
        painter.setBrush(self.colors['terminals'])
        painter.drawRect(30, 130, 40, 40)
        painter.drawRect(510, 130, 40, 40)
        
        # Draw FePO4 ions
        painter.setFont(self.fonts['fepo4'])
        for ion in self.fepo4_ions:
            painter.setBrush(self.colors['fepo4'])
            painter.setPen(Qt.NoPen)
            painter.drawEllipse(QPointF(ion['x'], ion['y']), ion['size'], ion['size'])
            
            painter.setPen(self.pens['fepo4_text'])
            painter.drawText(QRectF(ion['x']-12, ion['y']-4, 24, 8), Qt.AlignCenter, "FePO4")

        # Draw Li+ ions
        painter.setFont(self.fonts['ion'])
        for ion in self.ions:
            painter.setBrush(self.colors['ion'])
            painter.setPen(Qt.NoPen)
            painter.drawEllipse(QPointF(ion['x'], ion['y']), ion['size'], ion['size'])
            
            painter.setPen(self.pens['ion_text'])
            painter.drawText(QRectF(ion['x']-8, ion['y']-4, 16, 8), Qt.AlignCenter, "Li+")

        painter.setPen(self.colors['labels'])
        painter.setFont(self.fonts['labels'])
        painter.drawText(100, 250, "Anode (-)")
        painter.drawText(350, 250, "Cathode (+)")


class CSVPlotter(QWidget):
    def __init__(self):
        super().__init__()
        self.df = None
        self.init_ui()
        
    def init_ui(self):
        layout = QVBoxLayout()
        self.setLayout(layout)
        
        # Load button
        self.load_btn = QPushButton('Load CSV File')
        self.load_btn.clicked.connect(self.load_and_plot)
        layout.addWidget(self.load_btn)
        
        # Plot canvas
        self.figure = Figure(figsize=(6, 4))
        self.canvas = FigureCanvas(self.figure)
        layout.addWidget(self.canvas)
        
    def load_and_plot(self):
        file_path, _ = QFileDialog.getOpenFileName(self, 'Open CSV', '', 'CSV Files (*.csv)')
        
        if file_path:
            try:
                self.df = pd.read_csv(file_path)
                numeric_cols = self.df.select_dtypes(include=['number']).columns
                
                if len(numeric_cols) >= 2:
                    voltage_col = numeric_cols[0]
                    current_col = numeric_cols[1]
                    
                    self.figure.clear()
                    ax = self.figure.add_subplot(111)
                    ax.plot(self.df[voltage_col], self.df[current_col], 'b-', linewidth=2)
                    ax.set_xlabel('Voltage (V)')
                    ax.set_ylabel('Current (A)')
                    ax.set_title('Voltage vs Current')
                    ax.grid(True, alpha=0.3)
                    
                    self.figure.tight_layout()
                    self.canvas.draw()
                    
            except Exception as e:
                print(f"Error: {e}")


class MainWindow(QMainWindow):
    def __init__(self):
        super().__init__()
        self.init_ui()
        
    def init_ui(self):
        self.setWindowTitle('Battery Animation & CSV Plotter')
        self.setGeometry(100, 100, 1200, 800)
        
        # Main widget
        main_widget = QWidget()
        self.setCentralWidget(main_widget)
        
        # Main layout - vertical split
        main_layout = QVBoxLayout()
        main_widget.setLayout(main_layout)
        
        # Top section - Battery animation with controls
        battery_section = QWidget()
        battery_layout = QVBoxLayout()
        battery_section.setLayout(battery_layout)
        
        # Battery control buttons
        button_layout = QHBoxLayout()
        self.charge_btn = QPushButton('Start Charging')
        self.discharge_btn = QPushButton('Start Discharging')
        self.charge_btn.clicked.connect(self.start_charging)
        self.discharge_btn.clicked.connect(self.start_discharging)
        
        button_layout.addWidget(QLabel('Battery Controls:'))
        button_layout.addWidget(self.charge_btn)
        button_layout.addWidget(self.discharge_btn)
        button_layout.addStretch()
        
        # Battery widget
        self.battery_widget = BatteryWidget()
        
        battery_layout.addLayout(button_layout)
        battery_layout.addWidget(self.battery_widget)
        
        # Bottom section - CSV plotter
        csv_section = QWidget()
        csv_layout = QVBoxLayout()
        csv_section.setLayout(csv_layout)
        
        csv_layout.addWidget(QLabel('CSV Data Plotter:'))
        self.csv_plotter = CSVPlotter()
        csv_layout.addWidget(self.csv_plotter)
        
        # Add both sections to main layout
        main_layout.addWidget(battery_section)
        main_layout.addWidget(csv_section)
        
        # Set size proportions (battery takes less space)
        main_layout.setStretchFactor(battery_section, 1)
        main_layout.setStretchFactor(csv_section, 2)
    
    def start_charging(self):
        self.battery_widget.set_charging_mode(True)
        self.charge_btn.setStyleSheet("background-color: lightgreen;")
        self.discharge_btn.setStyleSheet("")
    
    def start_discharging(self):
        self.battery_widget.set_charging_mode(False)
        self.discharge_btn.setStyleSheet("background-color: lightcoral;")
        self.charge_btn.setStyleSheet("")


def main():
    app = QApplication(sys.argv)
    window = MainWindow()
    window.show()
    sys.exit(app.exec_())


if __name__ == '__main__':
    main()
