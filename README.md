踢球.py的代码：
#
import os
import sys
import sqlite3
import pandas as pd
from PyQt5.QtSql import QSqlDatabase

from db_init import create_views
from theme_manager import ThemeManager
from PyQt5.QtCore import QDate, Qt, pyqtSignal, QEvent, QSize
from datetime import datetime
from PyQt5.QtWidgets import (
    QApplication, QMainWindow, QWidget, QTableWidget, QTableWidgetItem,
    QPushButton, QVBoxLayout, QHBoxLayout, QLineEdit, QLabel,
    QDialog, QFormLayout, QMessageBox, QHeaderView, QFileDialog,
    QGroupBox, QMenu, QAction, QComboBox, QDateEdit, QGraphicsDropShadowEffect, QGridLayout
)
from PyQt5.QtGui import QFont, QColor, QIcon, QPainter, QPixmap, QPainterPath
from widgets.title_bar import TitleBar
from assets.styles.styles import LIGHT_THEME, DARK_THEME
class CoinSystem(QMainWindow):
    def closeEvent(self, event):
        if hasattr(self, "conn"):
            self.get_db_connection().close()
        event.accept()
#  主程序界面
class CoinSystem(QMainWindow):
    def __init__(self):
        super().__init__()
        # === 窗口初始化 ===
        self.setWindowFlags(Qt.FramelessWindowHint | Qt.Window)
        self.setAttribute(Qt.WA_TranslucentBackground)  # 设置窗口属性为透明——必须保留
        self.setWindowTitle("鸠灵")
        self.resize(1280, 720)
        self.setGeometry(100, 100, 1280, 720)
        # === 标题栏 ===
        # 创建唯一标题栏
        self.title_bar = TitleBar(self)
        self.setMenuWidget(self.title_bar)# 替换系统菜单栏
        for widget in self.findChildren(QWidget):
            if isinstance(widget, TitleBar) and widget != self.title_bar:
                widget.deleteLater()
        # === 主题管理 ===
        self.theme_manager = ThemeManager(self)  # 正确初始化主题管理器
        self.setStyleSheet(LIGHT_THEME)  # 初始应用白天主题
        #  连接信号
        self.title_bar.theme_btn.clicked.connect(self.theme_manager.toggle_theme)
        # === 数据库初始化 ===
        self.db_init()
        # === UI初始化 ===
        self.ui_init()

        # 添加背景相关属性
        self.background_image = None
        self.bg_opacity = 0.80  # 背景透明度
        self.set_initial_background()  #  加载默认背景

# === 数据库 ===
    def get_db_connection(self):
        if not hasattr(self, '_db_conn') or not self._db_conn:
            self._db_conn = sqlite3.connect('怎样的结局，配得上你这一路颠沛流离.db')
            self._db_conn.execute("PRAGMA journal_mode=WAL")
            self._db_conn.execute("PRAGMA synchronous = NORMAL")
            self._db_conn.execute("CREATE INDEX IF NOT EXISTS idx_daily_user ON daily_expenses(user_id)")
            self._db_conn.execute("CREATE INDEX IF NOT EXISTS idx_daily_date ON daily_expenses(date)")
        return self._db_conn
    def db_init(self):
        """初始化数据库连接和表结构"""
        self.get_db_connection().execute("PRAGMA journal_mode=WAL")  #  启用wal模式
        self.get_db_connection().execute("PRAGMA synchronous = NORMAL")
        self.get_db_connection().execute("PRAGMA cache_size = -10000")
        self.get_db_connection().execute("PRAGMA foreign_keys = ON")
        c = self.get_db_connection().cursor()
        self.create_tables()
        create_views(self.get_db_connection())
    def ui_init(self):
        """初始化界面组件"""
        # 主部件和布局
        main_widget = QWidget()
        main_widget.setObjectName("centralWidget")
        self.setCentralWidget(main_widget)
        # 用户界面组件初始化
        self.init_ui()
        self.load_users()
        self.enable_auto_save()
# === 样式更新方法 ===
    def update_styles(self, is_dark):
        """动态更新图标等需要重新着色的元素"""
        icon_color = "#FFFFFF" if is_dark else "#333333"
        # 更新标题栏图标
        self.title_bar.icon.setPixmap(
            self._colorize_icon("assets/icons/程序图标.svg", icon_color)
        )
    def enable_auto_save(self):
        # 启用表格编辑
        self.user_table.setEditTriggers(QTableWidget.DoubleClicked | QTableWidget.EditKeyPressed)
        # 连接单元格修改信号
        self.user_table.itemChanged.connect(self.on_user_edited)
    def _colorize_icon(self, path, color):
        """动态着色图标"""
        pixmap = QPixmap(path)
        painter = QPainter(pixmap)
        painter.setCompositionMode(QPainter.CompositionMode_SourceIn)
        painter.fillRect(pixmap.rect(), QColor(color))
        painter.end()
        return pixmap
    def on_user_edited(self, item):
        # 修改4：优化自动保存逻辑
        row = item.row()
        col = item.column()
        if col == 0:  # ID列不允许编辑
            return
        original_state = self.user_table.blockSignals(True)  # 阻塞信号
        try:
            user_id = int(self.user_table.item(row, 0).text())
            c = self.get_db_connection().cursor()
            if col == 1:  # 修改昵称
                new_name = item.text().strip()
                if not new_name:
                    raise ValueError("昵称不能为空")
                # 检查昵称唯一性
                c.execute('''
                    SELECT id FROM users 
                    WHERE nickname = ? AND id != ?
                ''', (new_name, user_id))
                if c.fetchone():
                    raise ValueError("昵称已存在")
                c.execute('''
                    UPDATE users SET nickname = ? 
                    WHERE id = ?
                ''', (new_name, user_id))
            elif col == 2:  # 修改存币
                try:
                    new_coins = int(item.text())
                except ValueError:
                    raise ValueError("存币必须为整数")
                if new_coins < 0:
                    raise ValueError("存币不能为负数")
                c.execute('''
                    UPDATE users SET total_coins = ? 
                    WHERE id = ?
                ''', (new_coins, user_id))
            self.get_db_connection().commit()
            self.load_users()  # 刷新数据
        except Exception as e:
            self.get_db_connection().rollback()
            QMessageBox.warning(self, "保存失败", str(e))
            self.load_users()  # 恢复原始值
        finally:
            self.user_table.blockSignals(original_state)  # 恢复信号
    def create_tables(self):
        c = self.get_db_connection().cursor()
        c.execute('''CREATE TABLE IF NOT EXISTS users
                     (id INTEGER PRIMARY KEY,
                      nickname TEXT UNIQUE,
                      total_coins INTEGER)''')
        # 修改表定义添加级联删除
        c.execute('''CREATE TABLE IF NOT EXISTS daily_expenses
                     (id INTEGER PRIMARY KEY,
                      user_id INTEGER,
                      date TEXT, 
                      flower INTEGER DEFAULT 0,
                      da_bai INTEGER DEFAULT 0,
                      cake INTEGER DEFAULT 0,
                      ring INTEGER DEFAULT 0,
                      crown INTEGER DEFAULT 0,
                      plane INTEGER DEFAULT 0,
                      rocket INTEGER DEFAULT 0,
                      FOREIGN KEY(user_id) REFERENCES users(id) ON DELETE CASCADE)''')  # 添加级联删除
        self.get_db_connection().commit()
    def delete_selected_user(self):
        selected = self.user_table.selectedItems()
        if not selected:
            return
        rows = set(item.row() for item in selected)
        uids = [self.user_table.item(row,0).text() for row in rows]
        reply = QMessageBox.question(self, '确认删除',
                                    f'确定要删除{len(uids)}位用户吗？')
        if reply == QMessageBox.Yes:
            c = self.get_db_connection().cursor()
            # 只需删除用户，消费记录会自动删除
            c.execute(f"DELETE FROM users WHERE id IN ({','.join(['?']*len(uids))})", uids)
            self.get_db_connection().commit()
            self.load_users()
        selected = self.user_table.selectedItems()
        if not selected:
            return
        rows = set(item.row() for item in selected)
        uids = [self.user_table.item(row,0).text() for row in rows]
        reply = QMessageBox.question(self, '确认删除',
                                    f'确定要删除{len(uids)}位用户及其所有消费记录吗？')
        if reply == QMessageBox.Yes:
            c = self.get_db_connection().cursor()
            c.execute(f"DELETE FROM users WHERE id IN ({','.join(['?']*len(uids))})", uids)
            c.execute(f"DELETE FROM daily_expenses WHERE user_id IN ({','.join(['?']*len(uids))})", uids)
            self.get_db_connection().commit()
            self.load_users()
    def batch_submit_expenses(self):
        try:
            c = self.get_db_connection().cursor()
            updated_users = []
            with self.get_db_connection():
                for row in range(self.expense_table.rowCount()):
                    # 2. 修正时间记录问题（修改batch_submit_expenses方法）
                    # 修改插入语句中的时间函数
                    c.execute('''INSERT INTO daily_expenses 
                                               (user_id, date, flower, da_bai, cake, ring, crown, plane, rocket)
                                               VALUES (?, datetime('now', 'localtime'), ?, ?, ?, ?, ?, ?, ?)''',
                              (uid, *expenses))
                    uid_item = self.expense_table.item(row, 0)
                    coins_item = self.expense_table.item(row, 2)
                    if not uid_item or not coins_item:
                        continue
                    uid = int(uid_item.text())
                    current_coins = int(coins_item.text())
                    expenses = []
                    for col in range(3, 10):
                        item = self.expense_table.item(row, col)
                        try:
                            value = int(item.text()) if item else 0
                            if value < 0:
                                raise ValueError("消费数量不能为负数")
                            expenses.append(value)
                        except ValueError:
                            expenses.append(0)
                    prices = [1, 20, 50, 180, 500, 1000, 2000]
                    total = sum(e * p for e, p in zip(expenses, prices))
                    if total == 0:
                        continue
                    c.execute('''INSERT INTO daily_expenses 
                               (user_id, date, flower, da_bai, cake, ring, crown, plane, rocket)
                               VALUES (?,  datetime('now'), ?, ?, ?, ?, ?, ?, ?)''',
                             (uid, *expenses))
                    new_coins = current_coins - total
                    if new_coins < 0:
                        QMessageBox.warning(self, "警告",
                                          f"用户 {self.expense_table.item(row, 1).text()} 的余额将变为负数！")
                    c.execute("UPDATE users SET total_coins = ? WHERE id=?",
                             (new_coins, uid))
                    updated_users.append(uid)
                if not updated_users:
                    QMessageBox.warning(self, "提示", "没有需要提交的消费记录")
                    return
            self.load_users()
            QMessageBox.information(
                self, "提交成功",
                f"成功更新 {len(updated_users)} 位用户的消费记录"
            )
        except Exception as e:
            QMessageBox.critical(self, "错误", str(e))
    def init_ui(self):
        # === 主窗口初始化 ===
        main_widget = QWidget()   # 中心对象
        main_widget.setObjectName("centralWidget")  # 必须与样式表匹配
        self.setCentralWidget(main_widget)
        # === 主布局 ===
        main_layout = QHBoxLayout(main_widget)  # 水平布局
        main_layout.setContentsMargins(15, 15, 15, 30)  # 增加边距
        main_layout.setSpacing(15)
        # === 左侧布局（用户管理） ===
        left_widget = QWidget()
        left_layout = QVBoxLayout(left_widget)
        left_layout.setContentsMargins(0, 0, 0, 0)
        left_layout.setSpacing(10)
        # ---- 用户管理组 ----
        user_mgmt_group = QGroupBox("用户管理")
        user_mgmt_group.setObjectName("userGroup")  # 样式表标识
        #  创建网格化布局
        user_mgmt_layout = QGridLayout()
        # 第一行：昵称相关
        user_mgmt_layout.addWidget(QLabel("昵称:"), 0, 0)
        self.new_user_input = QLineEdit()    # 名称输入框
        user_mgmt_layout.addWidget(self.new_user_input, 0, 1, 1, 2)  # 占2列
        self.add_user_btn = QPushButton("新增用户")
        self.add_user_btn.clicked.connect(self.add_user)
        user_mgmt_layout.addWidget(self.add_user_btn, 3, 1)
        self.import_user_btn = QPushButton("导入用户")
        self.import_user_btn.clicked.connect(lambda: self.import_data('users'))
        user_mgmt_layout.addWidget(self.import_user_btn, 3, 0)
        # 第二行：币量相关
        user_mgmt_layout.addWidget(QLabel("初始币量:"), 1, 0)
        self.new_coins_input = QLineEdit()
        user_mgmt_layout.addWidget(self.new_coins_input, 1, 1, 1, 2)
        refresh_btn = QPushButton("刷新数据")
        refresh_btn.clicked.connect(self.load_users)
        user_mgmt_layout.addWidget(refresh_btn, 4, 1)
        self.export_user_btn = QPushButton("导出用户")
        self.export_user_btn.clicked.connect(lambda: self.export_data('users'))
        user_mgmt_layout.addWidget(self.export_user_btn, 4, 0)
        # 布局参数设置
        user_mgmt_layout.setColumnStretch(1, 1)  # 输入框列占比更大
        user_mgmt_layout.setColumnStretch(2, 1)
        user_mgmt_layout.setHorizontalSpacing(5)
        user_mgmt_group.setLayout(user_mgmt_layout)
        # ---- 用户表格 ----
        self.user_table = QTableWidget()
        self.user_table.setObjectName("userTable")  # 样式表标识
        self.user_table.setAlternatingRowColors(True)
        self.user_table.setColumnCount(3)
        self.user_table.setHorizontalHeaderLabels(["ID", "昵称", "存币"])
        self.user_table.setColumnHidden(0, True)
        self.user_table.horizontalHeader().setSectionResizeMode(QHeaderView.Stretch)
        self.user_table.setContextMenuPolicy(Qt.CustomContextMenu)
        self.user_table.customContextMenuRequested.connect(self.show_user_context_menu)
        self.user_table.setSortingEnabled(True)
        self.user_table.horizontalHeader().setSortIndicator(2, Qt.DescendingOrder)# 默认按存币降序
        self.user_table.setSortingEnabled(False)  # 禁用实时排序
        # 组装左侧布局
        left_layout.addWidget(user_mgmt_group)
        left_layout.addWidget(self.user_table)
        # === 右侧布局（消费记录） ===
        right_widget = QWidget()
        right_layout = QVBoxLayout(right_widget)
        right_layout.setContentsMargins(0, 0, 0, 0)
        right_layout.setSpacing(15)
        # ---- 消费表格 ----
        self.expense_table = QTableWidget()
        self.expense_table.setObjectName("expenseTable")
        self.expense_table.setAlternatingRowColors(True)
        self.expense_table.setColumnCount(10)
        self.expense_table.setHorizontalHeaderLabels([
            "用户ID", "昵称", "存币", "花花 (1)", "大白 (20)",
            "蛋糕 (50)", "钻戒 (180)", "皇冠 (500)", "飞机 (1000)", "火箭 (2000)"
        ])
        self.expense_table.horizontalHeader().setSectionResizeMode(QHeaderView.Stretch)
        self.expense_table.setColumnHidden(0, True)
        self.expense_table.setSortingEnabled(True)
        # 底部按钮
        bottom_btn_widget = QWidget()
        bottom_btn_layout = QHBoxLayout(bottom_btn_widget)
        bottom_btn_layout.addStretch()
        self.history_btn = QPushButton("历史记录")
        self.history_btn.clicked.connect(self.show_history)
        self.batch_submit_btn = QPushButton("批量提交")
        self.report_btn = QPushButton("生成报表")
        self.batch_submit_btn.clicked.connect(self.batch_submit_expenses)
        self.report_btn.clicked.connect(self.show_report)
        bottom_btn_layout.addWidget(self.batch_submit_btn)
        bottom_btn_layout.addWidget(self.history_btn)
        bottom_btn_layout.addWidget(self.report_btn)
         # === 信号连接 ===
        self.user_table.itemChanged.connect(self.on_user_edited)
        self.title_bar.min_btn.clicked.connect(self.showMinimized)
        self.history_btn.clicked.connect(self.show_history)
        self.title_bar.close_btn.clicked.connect(self.close)
        # 组装右侧布局
        right_layout.addWidget(self.expense_table)
        main_layout.addWidget(left_widget, 30)
        right_layout.addWidget(bottom_btn_widget)
        main_layout.addWidget(right_widget, 70)
        self.set_style()
        # 设置默认背景
        self.set_initial_background()

    # === 背景设置 ===
    def set_initial_background(self):
        """设置默认背景"""
        default_bg = "assets/background/default.jpg"
        if os.path.exists(default_bg):
            self.background_image = QPixmap(default_bg)
        else:  # 添加备用背景
            self.background_image = QPixmap(400, 300)
            self.background_image.fill(QColor(240, 240, 240))
        self.update()

    def paintEvent(self, event):
        """重绘事件添加背景和圆角处理"""
        painter = QPainter(self)
        painter.setRenderHint(QPainter.Antialiasing)
        # 绘制背景
        if self.background_image:
            # 创建圆角矩形路径
            # path = QPainterPath()  #  圆角矩形路径导致主题切换
            # path.addRoundedRect(self.rect(), 12, 12)
            # 自适应缩放填充
            scaled_bg = self.background_image.scaled(
                self.size(),
                Qt.KeepAspectRatioByExpanding,
                Qt.SmoothTransformation
            )
            painter.setOpacity(self.bg_opacity)
            painter.drawPixmap(0, 0, scaled_bg)
        # 绘制圆角边框（仅在非最大化时）
        if not self.isMaximized():
            painter.setPen(Qt.NoPen)
            painter.setBrush(QColor(255, 255, 255, 20))
            painter.drawRoundedRect(self.rect(), 12, 12)
        painter.end()

    def change_background(self):
        """更换背景图片方法"""
        path, _ = QFileDialog.getOpenFileName(
            self, "选择背景图片", "",
            "图片文件 (*.jpg *.png *.bmp)")
        if path:
            self.background_image = QPixmap(path)
            self.update()  # 触发重绘

    # === 样式设置 ===
    def set_style(self):
        base_style = """
            QGroupBox#userGroup {
                border: 1px solid rgba(0,0,0,0.1);
            }
        """
        self.setStyleSheet(self.styleSheet() + base_style)
    # === 用户加载 ===
    def load_users(self):
        try:
            c = self.conn.cursor()
            c.execute("SELECT nickname FROM users")
            users = [row[0] for row in c.fetchall()]
            self.user_combo.addItems(users)
        except sqlite3.Error as e:
            print("加载用户失败:", e)

    # 事件过滤器
    def eventFilter(self, source, event):
        if event.type() == QEvent.MouseMove and source is self.user_table.viewport():
            item = self.user_table.itemAt(event.pos())
            if item: self.user_table.setCurrentCell(item.row(), item.column())
        return super().eventFilter(source, event)
    def show_user_context_menu(self, pos):
        menu = QMenu()
        delete_action = QAction("删除用户", self)
        delete_action.triggered.connect(self.delete_selected_user)
        menu.addAction(delete_action)
        menu.exec_(self.user_table.viewport().mapToGlobal(pos))
    def show_history(self):
        dialog = HistoryDialog(self.get_db_connection())
        dialog.data_deleted.connect(self.load_users)  # 连接信号到刷新方法
        dialog.exec_()
    def add_user(self):
        name = self.new_user_input.text()
        coins = self.new_coins_input.text()
        if not name or not coins.isdigit():
            QMessageBox.warning(self, "错误", "请输入有效的昵称和币量")
            return
        try:
            c = self.get_db_connection().cursor()
            c.execute("INSERT INTO users (nickname, total_coins) VALUES (?, ?)",
                      (name, int(coins)))
            self.get_db_connection().commit()
            self.load_users()
            self.new_user_input.clear()
            self.new_coins_input.clear()
        except sqlite3.IntegrityError:
            QMessageBox.warning(self, "错误", "该昵称已存在")
    def batch_submit_expenses(self):
        try:
            c = self.get_db_connection().cursor()
            updated_users = []
            with self.get_db_connection():
                for row in range(self.expense_table.rowCount()):
                    uid_item = self.expense_table.item(row, 0)
                    coins_item = self.expense_table.item(row, 2)
                    if not uid_item or not coins_item:
                        continue
                    uid = int(uid_item.text())
                    current_coins = int(coins_item.text())
                    # 获取消费数据
                    expenses = []
                    for col in range(3, 10):
                        item = self.expense_table.item(row, col)
                        try:
                            value = int(item.text()) if item else 0
                            if value < 0:
                                raise ValueError("消费数量不能为负数")
                            expenses.append(value)
                        except ValueError:
                            expenses.append(0)
                    # 计算总消费
                    prices = [1, 20, 50, 180, 500, 1000, 2000]
                    total = sum(e * p for e, p in zip(expenses, prices))
                    if total == 0:
                        continue  # 跳过无消费记录
                    # 更新消费记录
                    c.execute('''INSERT INTO daily_expenses 
                               (user_id, date, flower, da_bai, cake, ring, crown, plane, rocket)
                               VALUES (?, datetime('now', 'localtime'), ?, ?, ?, ?, ?, ?, ?)''',
                              (uid, *expenses))
                    # 更新用户币量
                    new_coins = current_coins - total
                    if new_coins < 0:
                        QMessageBox.warning(self, "警告",
                                            f"用户 {self.expense_table.item(row, 1).text()} 的余额将变为负数！")
                    c.execute("UPDATE users SET total_coins = ? WHERE id=?",
                              (new_coins, uid))
                    updated_users.append(uid)
                if not updated_users:
                    QMessageBox.warning(self, "提示", "没有需要提交的消费记录")
                    return
            # 刷新数据
            self.load_users()
            QMessageBox.information(
                self, "提交成功",
                f"成功更新 {len(updated_users)} 位用户的消费记录"
            )
        except ValueError as ve:
            QMessageBox.warning(self, "输入错误", str(ve))
        except Exception as e:
            QMessageBox.critical(self, "提交错误", f"发生未知错误：{str(e)}")
    def show_report(self):
        report_dialog = QDialog(self)
        report_dialog.setWindowTitle("消费报表")
        report_dialog.resize(800, 600)
        layout = QVBoxLayout()
        table = QTableWidget()
        table.setColumnCount(3)
        table.setHorizontalHeaderLabels(["用户", "总消费", "存币"])
        c = self.get_db_connection().cursor()
        c.execute('''SELECT u.nickname, 
                     SUM(d.flower*1 + d.da_bai*20 + d.cake*50 +
                         d.ring*180 + d.crown*500 + d.plane*1000 + d.rocket*2000),
                     u.total_coins
                  FROM users u
                  LEFT JOIN daily_expenses d ON u.id = d.user_id
                  GROUP BY u.id''')
        data = c.fetchall()
        table.setRowCount(len(data))
        for row, (name, spent, remaining) in enumerate(data):
            table.setItem(row, 0, QTableWidgetItem(name))
            table.setItem(row, 1, QTableWidgetItem(str(spent or 0)))
            table.setItem(row, 2, QTableWidgetItem(str(remaining)))
            if remaining < 0:
                table.item(row, 2).setBackground(QColor("#ff7675"))
        layout.addWidget(table)
        report_dialog.setLayout(layout)
        report_dialog.exec_()
    def export_data(self, data_type):
        try:
            if data_type == 'expenses':
                # 修改查询语句显示用户名
                query = '''
                    SELECT u.nickname as 用户,
                           strftime('%Y-%m-%d %H:%M:%S', d.date) as 时间,
                           d.flower as 花花,
                           d.da_bai as 大白,
                           d.cake as 蛋糕,
                           d.ring as 钻戒, 
                           d.crown as 皇冠,
                           d.plane as 飞机,
                           d.rocket as 火箭
                    FROM daily_expenses d
                    JOIN users u ON d.user_id = u.id'''
                df = pd.read_sql_query(query, self.get_db_connection())
            path, _ = QFileDialog.getSaveFileName(
                self, "保存文件", "", "Excel文件 (*.xlsx)")
            if not path:
                return
            with pd.ExcelWriter(path) as writer:
                if data_type == 'users':
                    df = pd.read_sql("SELECT * FROM users", self.get_db_connection())
                    df.to_excel(writer, index=False)
                else:
                    df = pd.read_sql_query(query, self.get_db_connection())
                    df.to_excel(writer, index=False)
            QMessageBox.information(self, "成功", "导出完成")
        except Exception as e:
            QMessageBox.critical(self, "错误", str(e))
    def import_data(self, data_type):
        try:
            path, _ = QFileDialog.getOpenFileName(
                self, "选择文件", "", "Excel文件 (*.xlsx)")
            if not path:
                return
            df = pd.read_excel(path)
            with self.get_db_connection():
                c = self.get_db_connection().cursor()
                if data_type == 'users':
                    for _, row in df.iterrows():
                        c.execute('''INSERT OR REPLACE INTO users 
                                   (id, nickname, total_coins) VALUES (?, ?, ?)''',
                                 (row['id'], row['nickname'], row['total_coins']))
                else:
                    for _, row in df.iterrows():
                        c.execute('''INSERT OR REPLACE INTO daily_expenses 
                                   (id, user_id, date, flower, da_bai, cake, ring, crown, plane, rocket)
                                   VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)''',
                                 (row['id'], row['user_id'], row['date'],
                                  row.get('flower',0), row.get('da_bai',0),
                                  row.get('cake',0), row.get('ring',0),
                                  row.get('crown',0), row.get('plane',0),
                                  row.get('rocket',0)))
            self.load_users()
            QMessageBox.information(self, "成功", "导入完成")
        except Exception as e:
            QMessageBox.critical(self, "错误", str(e))
# 历史消费记录的窗口
class HistoryDialog(QDialog):
    data_deleted = pyqtSignal()  # 新增信号
    def __init__(self, conn):
        super().__init__()
        self.conn = conn
        self.setWindowTitle("历史消费记录")
        self.resize(1200, 600)
        self.page_size = 100  # 新增分页
        self.current_page = 0
        self.total_pages = 0
        self.init_ui()
        self.init_filter_ui()
        self.load_data()
    def init_ui(self):
        layout = QVBoxLayout(self)
        # 表格
        self.table = QTableWidget()
        self.table.setColumnCount(7)
        self.table.setHorizontalHeaderLabels([
            "ID", "用户", "时间", "项目", "数量", "单价", "总金额"
        ])
        self.table.horizontalHeader().setSectionResizeMode(QHeaderView.Stretch)
        # 分页控件
        pagination = QHBoxLayout()
        self.prev_btn = QPushButton("上一页")
        self.next_btn = QPushButton("下一页")
        self.page_label = QLabel()
        pagination.addWidget(self.prev_btn)
        pagination.addWidget(self.page_label)
        pagination.addWidget(self.next_btn)
        # 按钮
        btn_layout = QHBoxLayout()
        self.delete_btn = QPushButton("删除记录")
        self.export_btn = QPushButton("导出Excel")
        self.import_btn = QPushButton("导入Excel")
        btn_layout.addWidget(self.delete_btn)
        btn_layout.addWidget(self.export_btn)
        btn_layout.addWidget(self.import_btn)
        # 事件绑定
        self.prev_btn.clicked.connect(self.prev_page)
        self.next_btn.clicked.connect(self.next_page)
        self.delete_btn.clicked.connect(self.delete_selected)
        self.export_btn.clicked.connect(self.export_data)
        self.import_btn.clicked.connect(self.import_data)
        # 组装布局
        layout.addLayout(pagination)
        layout.addWidget(self.table)
        layout.addLayout(btn_layout)

    # 内存管理强化
    def closeEvent(self, event):
        self.model.clear()
        self.db.close()
        QSqlDatabase.removeDatabase("history_view")
        event.accept()

    def __del__(self):
        if hasattr(self, 'model'):
            self.model.deleteLater()
    def load_data(self):
        try:
            date_str = self.date_edit.date().toString("yyyy-MM-dd")
            user_filter = ""
            params = [date_str + "%"]
            if self.user_combo.currentText() != "所有用户":
                user_filter = " AND u.nickname = ?"
                params.append(self.user_combo.currentText())
            # 计算总数
            count_sql = f"""
                SELECT COUNT(*) 
                FROM expanded_data_view 
                WHERE date LIKE ? {user_filter}
            """
            c = self.conn.cursor()
            c.execute(count_sql, params)
            total = c.fetchone()[0]
            self.total_pages = (total + self.page_size - 1) // self.page_size
            # 加载数据
            data_sql = f"""
                SELECT * 
                FROM expanded_data_view 
                WHERE date LIKE ? {user_filter}
                ORDER BY date DESC
                LIMIT ? OFFSET ?
            """
            page_params = params + [self.page_size, self.current_page * self.page_size]
            c.execute(data_sql, page_params)
            self.table.setRowCount(0)
            for row_idx, row_data in enumerate(c.fetchall()):
                self.table.insertRow(row_idx)
                for col_idx, col_data in enumerate(row_data):
                    item = QTableWidgetItem(str(col_data))
                    if col_idx in [4,5,6]:  # 数值列右对齐
                        item.setTextAlignment(Qt.AlignRight | Qt.AlignVCenter)
                    self.table.setItem(row_idx, col_idx, item)
            self.update_page_label()
        except sqlite3.Error as e:
            QMessageBox.critical(self, "数据库错误", str(e))

    def update_page_label(self):
        self.page_label.setText(f"第 {self.current_page + 1} 页 / 共 {self.total_pages} 页")
        self.prev_btn.setEnabled(self.current_page > 0)
        self.next_btn.setEnabled(self.current_page < self.total_pages - 1)
    def prev_page(self):
        if self.current_page > 0:
            self.current_page -= 1
            self.load_data()
    def next_page(self):
        if self.current_page < self.total_pages - 1:
            self.current_page += 1
            self.load_data()
    # 修改点1：增强筛选UI初始化
    def init_filter_ui(self):
        # 创建筛选工具栏
        filter_widget = QWidget()
        filter_layout = QHBoxLayout(filter_widget)
        # 日期筛选控件
        self.date_edit = QDateEdit()
        self.date_edit.setDate(QDate.currentDate())
        self.date_edit.dateChanged.connect(self.load_data)
        # 用户筛选
        self.user_combo = QComboBox()
        self.user_combo.addItem("所有用户")
        self.load_users()
        self.user_combo.currentIndexChanged.connect(self.load_data)
        filter_layout.addWidget(QLabel("日期:"))
        filter_layout.addWidget(self.date_edit)
        filter_layout.addWidget(QLabel("用户:"))
        filter_layout.addWidget(self.user_combo)
        self.layout().insertWidget(0, filter_widget)
    # 修改点2：增强数据加载
    def load_data(self):
        """加载数据并应用筛选条件"""
        try:
            self.table.setUpdatesEnabled(False)  # 禁用表格更新
            conditions = []
            params = []
            # 日期筛选（使用字符串截取）
            selected_date = self.date_edit.date().toString("yyyy-MM-dd")
            conditions.append("substr(expanded_data.date, 1, 10) = ?")
            params.append(selected_date)
            # 用户筛选（正确处理“所有用户”）
            selected_user = self.user_combo.currentText()
            if selected_user and selected_user != "所有用户":
                conditions.append("expanded_data.nickname = ?")
                params.append(selected_user)
            # 构建基础查询
            base_query = '''
            WITH expanded_data AS (
                SELECT d.id, u.nickname, 
                    strftime('%Y-%m-%d %H:%M:%S', d.date) as date, 
                    '花花' as item, flower as quantity, 1 as price, flower*1 as total
                FROM daily_expenses d
                JOIN users u ON d.user_id = u.id WHERE flower > 0
                UNION ALL
                SELECT d.id, u.nickname, strftime('%Y-%m-%d %H:%M:%S', d.date), 
                       '大白', da_bai, 20, da_bai*20 FROM daily_expenses d JOIN users u ON d.user_id = u.id WHERE da_bai > 0
                UNION ALL
                SELECT d.id, u.nickname, strftime('%Y-%m-%d %H:%M:%S', d.date), 
                       '蛋糕', cake, 50, cake*50 FROM daily_expenses d JOIN users u ON d.user_id = u.id WHERE cake > 0
                UNION ALL
                SELECT d.id, u.nickname, strftime('%Y-%m-%d %H:%M:%S', d.date), 
                       '钻戒', ring, 180, ring*180 FROM daily_expenses d JOIN users u ON d.user_id = u.id WHERE ring > 0
                UNION ALL
                SELECT d.id, u.nickname, strftime('%Y-%m-%d %H:%M:%S', d.date), 
                       '皇冠', crown, 500, crown*500 FROM daily_expenses d JOIN users u ON d.user_id = u.id WHERE crown > 0
                UNION ALL
                SELECT d.id, u.nickname, strftime('%Y-%m-%d %H:%M:%S', d.date), 
                       '飞机', plane, 1000, plane*1000 FROM daily_expenses d JOIN users u ON d.user_id = u.id WHERE plane > 0
                UNION ALL
                SELECT d.id, u.nickname, strftime('%Y-%m-%d %H:%M:%S', d.date), 
                       '火箭', rocket, 2000, rocket*2000 FROM daily_expenses d JOIN users u ON d.user_id = u.id WHERE rocket > 0
            )
            SELECT * FROM expanded_data '''
            # 添加筛选条件
            if conditions:
                base_query += ' WHERE ' + ' AND '.join(conditions)
            base_query += ' ORDER BY date DESC'
            # 执行查询并填充表格
            df = pd.read_sql_query(base_query, self.get_db_connection(), params=params)
            self.table.setRowCount(len(df))
            for row in range(len(df)):
                for col in range(7):
                    item = QTableWidgetItem(str(df.iat[row, col]))
                    if col in [4, 5, 6]:
                        item.setTextAlignment(Qt.AlignRight | Qt.AlignVCenter)
                    self.table.setItem(row, col, item)
        except Exception as e:
            QMessageBox.critical(self, "错误", f"加载失败: {str(e)}")
        finally:
            self.table.setUpdatesEnabled(True)  #  恢复更新

    def show_context_menu(self, pos):
        menu = QMenu()
        delete_action = QAction("删除记录", self)
        delete_action.triggered.connect(self.delete_selected)
        menu.addAction(delete_action)
        menu.exec_(self.table.viewport().mapToGlobal(pos))
    def delete_selected(self):
        # 修改2：删除时返还存币
        selected = set(index.row() for index in self.table.selectedIndexes())
        if not selected:
            return
        reply = QMessageBox.question(
            self, '确认删除',
            f'确定要删除{len(selected)}条记录吗？该操作不可逆！'
        )
        if reply != QMessageBox.Yes:
            return
        item_to_column = {
            '花花': 'flower',
            '大白': 'da_bai',
            '蛋糕': 'cake',
            '钻戒': 'ring',
            '皇冠': 'crown',
            '飞机': 'plane',
            '火箭': 'rocket'
        }
        try:
            c = self.get_db_connection().cursor()
            total_refund = 0
            deleted_ids = set()
            for row in selected:
                # 获取行数据
                id_item = self.table.item(row, 0)
                item_item = self.table.item(row, 3)
                qty_item = self.table.item(row, 4)
                price_item = self.table.item(row, 5)
                if not all([id_item, item_item, qty_item, price_item]):
                    continue
                record_id = int(id_item.text())
                item_name = item_item.text()
                quantity = int(qty_item.text())
                price = int(price_item.text())
                # 获取数据库字段名
                column = item_to_column.get(item_name)
                if not column:
                    continue
                # 获取用户ID和原始数据
                c.execute("SELECT user_id, {0} FROM daily_expenses WHERE id=?".format(column), (record_id,))
                result = c.fetchone()
                if not result:
                    continue
                user_id, original_qty = result
                # 校验数量一致性
                if original_qty != quantity:
                    QMessageBox.warning(self, "警告",
                        f"记录已变更！当前{item_name}数量为{original_qty}，无法执行删除")
                    continue
                # 计算返还金额
                refund = quantity * price
                total_refund += refund
                # 更新消费记录
                c.execute(f"UPDATE daily_expenses SET {column}=0 WHERE id=?", (record_id,))
                # 检查是否所有项目都已清零
                c.execute('''SELECT flower+da_bai+cake+ring+crown+plane+rocket 
                           FROM daily_expenses WHERE id=?''', (record_id,))
                total = c.fetchone()[0]
                if total == 0:
                    deleted_ids.add(record_id)
                # 更新用户余额
                c.execute("UPDATE users SET total_coins=total_coins+? WHERE id=?",
                        (refund, user_id))
            # 清理空记录
            if deleted_ids:
                c.execute(f"DELETE FROM daily_expenses WHERE id IN ({','.join(['?']*len(deleted_ids))})",
                         list(deleted_ids))
            self.get_db_connection().commit()
            self.load_data()
            self.data_deleted.emit()
            QMessageBox.information(self, "成功",
                f"已删除{len(selected)}条记录，共返还{total_refund}存币")
        except sqlite3.Error as e:
            self.get_db_connection().rollback()
            QMessageBox.critical(self, "数据库错误", str(e))
        except Exception as e:
            QMessageBox.critical(self, "系统错误", str(e))

    def export_data(self):
        path, _ = QFileDialog.getSaveFileName(
            self, "保存文件", "", "Excel文件 (*.xlsx)")
        if not path: return
        try:
            df = pd.read_sql("SELECT * FROM expanded_data_view", self.conn)
            df.to_excel(path, index=False)
            QMessageBox.information(self, "成功", "导出完成")
        except Exception as e:
            QMessageBox.critical(self, "错误", str(e))

    def import_data(self):
        path, _ = QFileDialog.getOpenFileName(
            self, "选择文件", "", "Excel文件 (*.xlsx)")
        if not path: return
        try:
            df = pd.read_excel(path)
            with self.conn:
                c = self.conn.cursor()
                for _, row in df.iterrows():
                    c.execute("""
                        INSERT INTO daily_expenses 
                        (user_id, date, flower, da_bai, cake, ring, crown, plane, rocket)
                        VALUES (
                            (SELECT id FROM users WHERE nickname = ?),
                            ?,
                            CASE WHEN ? = '花花' THEN ? ELSE 0 END,
                            CASE WHEN ? = '大白' THEN ? ELSE 0 END,
                            CASE WHEN ? = '蛋糕' THEN ? ELSE 0 END,
                            CASE WHEN ? = '钻戒' THEN ? ELSE 0 END,
                            CASE WHEN ? = '皇冠' THEN ? ELSE 0 END,
                            CASE WHEN ? = '飞机' THEN ? ELSE 0 END,
                            CASE WHEN ? = '火箭' THEN ? ELSE 0 END
                        )
                    """, (row['user'], row['date'],
                          row['item'], row['quantity'],
                          row['item'], row['quantity'],
                          row['item'], row['quantity'],
                          row['item'], row['quantity'],
                          row['item'], row['quantity'],
                          row['item'], row['quantity'],
                          row['item'], row['quantity']))
                self.load_data()
                QMessageBox.information(self, "成功", "导入完成")
        except Exception as e:
            QMessageBox.critical(self, "错误", str(e))

if __name__ == "__main__":
    app = QApplication(sys.argv)
    app.setStyle("Fusion")  # 重要：使用Fusion样式确保跨平台一致性
    window = CoinSystem()
    window.show()
    sys.exit(app.exec_())

db_init.py的代码：
#  数据库视图创建


def create_views(conn):
    c = conn.cursor()
    c.execute("""
    CREATE VIEW IF NOT EXISTS expanded_data_view AS
    SELECT
        d.id,
        u.nickname as user,
        strftime('%Y-%m-%d %H:%M:%S', d.date) as date,
        CASE
            WHEN d.flower > 0 THEN '花花'
            WHEN d.da_bai > 0 THEN '大白'
            WHEN d.cake > 0 THEN '蛋糕'
            WHEN d.ring > 0 THEN '钻戒'
            WHEN d.crown > 0 THEN '皇冠'
            WHEN d.plane > 0 THEN '飞机'
            WHEN d.rocket > 0 THEN '火箭'
        END as item,
        CASE
            WHEN d.flower > 0 THEN d.flower
            WHEN d.da_bai > 0 THEN d.da_bai
            WHEN d.cake > 0 THEN d.cake
            WHEN d.ring > 0 THEN d.ring
            WHEN d.crown > 0 THEN d.crown
            WHEN d.plane > 0 THEN d.plane
            WHEN d.rocket > 0 THEN d.rocket
        END as quantity,
        CASE
            WHEN d.flower > 0 THEN 1
            WHEN d.da_bai > 0 THEN 20
            WHEN d.cake > 0 THEN 50
            WHEN d.ring > 0 THEN 180
            WHEN d.crown > 0 THEN 500
            WHEN d.plane > 0 THEN 1000
            WHEN d.rocket > 0 THEN 2000
        END as price,
        (CASE
            WHEN d.flower > 0 THEN d.flower*1
            WHEN d.da_bai > 0 THEN d.da_bai*20
            WHEN d.cake > 0 THEN d.cake*50
            WHEN d.ring > 0 THEN d.ring*180
            WHEN d.crown > 0 THEN d.crown*500
            WHEN d.plane > 0 THEN d.plane*1000
            WHEN d.rocket > 0 THEN d.rocket*2000
        END) as total
    FROM daily_expenses d
    JOIN users u ON d.user_id = u.id
    WHERE d.flower + d.da_bai + d.cake + d.ring + d.crown + d.plane + d.rocket > 0
    """)
    conn.commit()


存在的问题

Traceback (most recent call last):
  File "F:\学习_python类\python_py\踢球\57_踢球3.7版_按钮调整.py", line 925, in <module>
    window = CoinSystem()
  File "F:\学习_python类\python_py\踢球\57_踢球3.7版_按钮调整.py", line 58, in __init__
    self.ui_init()
  File "F:\学习_python类\python_py\踢球\57_踢球3.7版_按钮调整.py", line 91, in ui_init
    self.load_users()
  File "F:\学习_python类\python_py\踢球\57_踢球3.7版_按钮调整.py", line 417, in load_users
    c = self.conn.cursor()
AttributeError: 'CoinSystem' object has no attribute 'conn'

进程已结束,退出代码1