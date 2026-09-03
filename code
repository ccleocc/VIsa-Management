
import sqlite3
from datetime import date, datetime, timedelta
from pathlib import Path
import pandas as pd
import streamlit as st

APP_DIR = Path(__file__).resolve().parent
DB_PATH = APP_DIR / "visa_reminder.db"

# 印尼移民规则（截至本版本说明）
# 1) ITK: 印尼移民局官方页面列明，Single-Entry/Multiple-Entry Visa Kunjungan
#    的停留许可可从初始 60 天延长，累计最长 180 天；VoA B 类初始 30 天，
#    B 类可延长 1 次至累计 60 天。
# 2) ITAS: 对期限不超过 1 年的 ITAS，续签申请最早可在到期前 30 天提出；
#    超过 1 年的 ITAS，最早可在到期前 3 个月提出；最晚均为原 ITAS 到期日。
#    如果在到期前已完成申请并缴费，即使审批跨过原到期日，一般不计为 overstay。
#
# 注意：具体 Visa index（如 C1/C2、B1/B2、E23 等）及活动权限可能随法规/移民局系统调整。
# 本程序因此保存 visa_index、许可类型和实际停留许可到期日，并允许管理员修改规则。

DEFAULT_RULES = [
    # code, visa_family, description, legal_window_start_days, legal_latest_days_before,
    # recommended_buffer_days, max_total_stay_days, extension_count, source_note
    ("B1", "ITK/VoA", "Visa Kunjungan Saat Kedatangan - wisata/MICE", 0, 0, 14, 60, 1,
     "官方移民局：首次最多30天；可延长1次，总停留最多60天。"),
    ("F1", "ITK/VoA", "Visa Kunjungan Saat Kedatangan - 相关访问", 0, 0, 14, 60, 1,
     "官方移民局/地区移民局服务页规则，具体以实际签发的许可为准。"),
    ("C", "ITK", "Visa Kunjungan单次/多次 - 访问类", 60, 0, 30, 180, 2,
     "官方地区移民局页面：初始60天，可延长至累计180天。"),
    ("C1", "ITK", "Visa Kunjungan - wisata", 60, 0, 30, 180, 2,
     "按 Visa Kunjungan/ITK 规则跟踪；请以实际签发的Izin Tinggal为准。"),
    ("C2", "ITK", "Visa Kunjungan - bisnis", 60, 0, 30, 180, 2,
     "业务访问类；具体权限和停留期以签证及Izin Tinggal实际文件为准。"),
    ("D", "ITK", "Visa Kunjungan多次", 60, 0, 30, 180, 2,
     "官方地区移民局页面：单/多次访问签对应ITK可延长至累计180天。"),
    ("E23", "ITAS", "Visa Tinggal Terbatas - 工作", 30, 0, 45, None, None,
     "官方移民局：ITAS≤1年，最早到期前30天申请；最晚至到期日。"),
    ("E24", "ITAS", "Visa Tinggal Terbatas - 数字/技术类工作", 30, 0, 45, None, None,
     "ITAS≤1年规则；实际时长取决于签发的ITAS。"),
    ("E25", "ITAS", "Visa Tinggal Terbatas - 管理/高管/董事等工作", 30, 0, 45, None, None,
     "ITAS≤1年规则；若ITAS期限>1年，系统会按3个月最早申请窗口提醒。"),
    ("E23/E24/E25", "ITAS", "工作ITAS（通用）", 30, 0, 45, None, None,
     "按ITAS实际期限自动判断：≤1年提前30天进入合法申请窗口；>1年提前90天。"),
]

def db():
    conn = sqlite3.connect(DB_PATH)
    conn.row_factory = sqlite3.Row
    return conn

def init_db():
    conn = db()
    conn.executescript("""
    CREATE TABLE IF NOT EXISTS employees (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        employee_no TEXT UNIQUE NOT NULL,
        name TEXT NOT NULL,
        department TEXT,
        passport_no TEXT,
        nationality TEXT DEFAULT 'CHN',
        visa_label TEXT NOT NULL,
        visa_index TEXT,
        permit_type TEXT NOT NULL DEFAULT 'ITK',
        visa_issue_date TEXT,
        visa_expiry_date TEXT,
        stay_permit_start_date TEXT,
        stay_permit_expiry_date TEXT NOT NULL,
        last_entry_date TEXT,
        extension_count INTEGER DEFAULT 0,
        sponsor TEXT,
        responsible_person TEXT,
        notes TEXT,
        active INTEGER DEFAULT 1,
        created_at TEXT,
        updated_at TEXT
    );

    CREATE TABLE IF NOT EXISTS rules (
        code TEXT PRIMARY KEY,
        visa_family TEXT NOT NULL,
        description TEXT,
        earliest_start_days INTEGER NOT NULL,
        legal_latest_before_days INTEGER NOT NULL DEFAULT 0,
        recommended_buffer_days INTEGER NOT NULL DEFAULT 30,
        max_total_stay_days INTEGER,
        extension_count INTEGER,
        source_note TEXT
    );

    CREATE TABLE IF NOT EXISTS status_rules (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        status_name TEXT NOT NULL,
        min_days INTEGER,
        max_days INTEGER,
        display_order INTEGER NOT NULL DEFAULT 100,
        description TEXT
    );
    """)

    # Visa / 政策规则只在表为空时初始化，防止重复增加。
    if conn.execute("SELECT COUNT(*) AS c FROM rules").fetchone()["c"] == 0:
        conn.executemany("""
        INSERT INTO rules
        (code,visa_family,description,earliest_start_days,legal_latest_before_days,
         recommended_buffer_days,max_total_stay_days,extension_count,source_note)
        VALUES (?,?,?,?,?,?,?,?,?)
        """, DEFAULT_RULES)

    # 到期状态固定5条。启动时直接重建，清理旧版本已经产生的重复项。
    default_status = [
        ("已过期", None, -1, 1, "到期日已过去"),
        ("紧急", 0, 7, 2, "距离到期7天以内"),
        ("重点关注", 8, 30, 3, "距离到期8-30天"),
        ("即将到期", 31, 60, 4, "距离到期31-60天"),
        ("正常", 61, None, 5, "距离到期61天以上"),
    ]

    status_count = conn.execute(
        "SELECT COUNT(*) AS c FROM status_rules"
    ).fetchone()["c"]

    if status_count == 0:
        conn.executemany("""
            INSERT INTO status_rules
            (status_name,min_days,max_days,display_order,description)
            VALUES (?,?,?,?,?)
        """, default_status)


    conn.commit()
    conn.close()


def parse_date(x):
    if x is None or str(x).strip() == "":
        return None
    if isinstance(x, date):
        return x
    return datetime.strptime(str(x)[:10], "%Y-%m-%d").date()

def fmt_date(x):
    d = parse_date(x)
    return d.strftime("%Y-%m-%d") if d else ""

def get_rule(visa_index):
    conn = db()
    row = conn.execute("SELECT * FROM rules WHERE code=?", (visa_index,)).fetchone()
    if not row:
        # 优先匹配通用规则
        row = conn.execute("SELECT * FROM rules WHERE code='E23/E24/E25'").fetchone() \
            if visa_index in {"E23","E24","E25"} else None
    conn.close()
    return dict(row) if row else None

def get_employees(active_only=False):
    conn = db()
    sql = "SELECT * FROM employees"
    if active_only:
        sql += " WHERE active=1"
    sql += " ORDER BY stay_permit_expiry_date ASC, name ASC"
    rows = conn.execute(sql).fetchall()
    conn.close()
    return pd.DataFrame([dict(r) for r in rows])

def calc_info(row, today=None):
    today = today or date.today()
    expiry = parse_date(row["stay_permit_expiry_date"])
    if not expiry:
        return {}
    days_left = (expiry - today).days

    permit_type = row.get("permit_type", "ITK")
    visa_index = row.get("visa_index") or ""
    rule = get_rule(visa_index)

    # ITAS: 官方申请窗口取决于“本次ITAS期限”是否 <= 1年。
    if permit_type.upper() == "ITAS":
        start = parse_date(row.get("stay_permit_start_date"))
        if start:
            duration = (expiry - start).days
            earliest_days = 30 if duration <= 366 else 90
            legal_earliest = expiry - timedelta(days=earliest_days)
        else:
            earliest_days = 30
            legal_earliest = expiry - timedelta(days=30)
        recommended_latest = expiry - timedelta(days=45 if earliest_days == 30 else 90)
        if recommended_latest < legal_earliest:
            recommended_latest = legal_earliest
        legal_latest = expiry
    else:
        # ITK / visitor: 程序将“建议最晚办理日”设置为到期前30天（VoA默认为14天），
        # 作为内部风险控制，不将其误写成法律强制期限。
        buffer_days = rule["recommended_buffer_days"] if rule else 30
        recommended_latest = expiry - timedelta(days=buffer_days)
        legal_earliest = None
        legal_latest = expiry

    status = get_status_name(days_left)

    return {
        "days_left": days_left,
        "status": status,
        "legal_earliest_date": legal_earliest,
        "legal_latest_date": legal_latest,
        "recommended_latest_date": recommended_latest,
        "rule": rule,
    }


def get_status_rules():
    conn = db()
    rows = conn.execute("""
        SELECT id,status_name,min_days,max_days,display_order,description
        FROM status_rules
        ORDER BY display_order ASC
    """).fetchall()
    conn.close()
    return [dict(r) for r in rows]

def get_status_name(days_left):
    for r in get_status_rules():
        min_d = r["min_days"]
        max_d = r["max_days"]
        if (min_d is None or days_left >= min_d) and (max_d is None or days_left <= max_d):
            return r["status_name"]
    return "未匹配规则"

def save_status_rules(rows):
    if len(rows) != 5:
        raise ValueError("到期状态判定规则固定为5条，只能修改，不能新增或删除。")

    cleaned = []
    names = set()
    for idx, r in enumerate(rows, start=1):
        name = str(r.get("status_name") or "").strip()
        if not name:
            raise ValueError(f"第{idx}行状态名称不能为空。")
        if name in names:
            raise ValueError(f"状态名称重复：{name}")
        names.add(name)

        def to_int(v):
            if v is None or (isinstance(v, float) and pd.isna(v)) or str(v).strip() == "":
                return None
            return int(v)

        min_d = to_int(r.get("min_days"))
        max_d = to_int(r.get("max_days"))

        if min_d is not None and max_d is not None and min_d > max_d:
            raise ValueError(f"{name}：最小天数不能大于最大天数。")

        cleaned.append((name, min_d, max_d, idx, str(r.get("description") or "").strip()))

    # 检查区间是否重叠。
    ordered = sorted(cleaned, key=lambda x: -10**9 if x[1] is None else x[1])
    for i in range(len(ordered) - 1):
        a, b = ordered[i], ordered[i + 1]
        if a[2] is None or b[1] is None or a[2] >= b[1]:
            raise ValueError(f"状态区间重叠：{a[0]} 与 {b[0]}。")

    conn = db()
    try:
        conn.execute("BEGIN")
        conn.execute("DELETE FROM status_rules")
        conn.executemany("""
            INSERT INTO status_rules
            (status_name,min_days,max_days,display_order,description)
            VALUES (?,?,?,?,?)
        """, cleaned)
        conn.commit()
    except Exception:
        conn.rollback()
        raise
    finally:
        conn.close()

def save_policy_rules(rows):
    cleaned = []
    seen = set()

    for r in rows:
        code = str(r.get("code") or "").strip()
        family = str(r.get("visa_family") or "").strip()

        # 动态编辑器中的完全空白新行忽略。
        if not code and not family:
            continue

        if not code or not family:
            raise ValueError("新增签证类别时，Visa Index 和 Family 不能为空。")
        if code in seen:
            raise ValueError(f"Visa Index 重复：{code}")
        seen.add(code)

        def to_int_or_none(v):
            if v is None or (isinstance(v, float) and pd.isna(v)) or str(v).strip() == "":
                return None
            return int(v)

        cleaned.append((
            code,
            family,
            str(r.get("description") or "").strip(),
            to_int_or_none(r.get("earliest_start_days")) or 0,
            to_int_or_none(r.get("legal_latest_before_days")) or 0,
            to_int_or_none(r.get("recommended_buffer_days")) or 0,
            to_int_or_none(r.get("max_total_stay_days")),
            to_int_or_none(r.get("extension_count")),
            str(r.get("source_note") or "").strip()
        ))

    if not cleaned:
        raise ValueError("至少保留一条签证类别规则。")

    conn = db()
    try:
        conn.execute("BEGIN")
        conn.execute("DELETE FROM rules")
        conn.executemany("""
            INSERT INTO rules
            (code,visa_family,description,earliest_start_days,legal_latest_before_days,
             recommended_buffer_days,max_total_stay_days,extension_count,source_note)
            VALUES (?,?,?,?,?,?,?,?,?)
        """, cleaned)
        conn.commit()
    except Exception:
        conn.rollback()
        raise
    finally:
        conn.close()


def add_or_update_employee(data, employee_id=None):
    conn = db()
    now = datetime.now().isoformat(timespec="seconds")
    fields = [
        "employee_no","name","department","passport_no","nationality","visa_label","visa_index",
        "permit_type","visa_issue_date","visa_expiry_date","stay_permit_start_date",
        "stay_permit_expiry_date","last_entry_date","extension_count","sponsor",
        "responsible_person","notes","active","updated_at"
    ]
    vals = [data.get(f) for f in fields]
    if employee_id:
        sets = ", ".join(f"{f}=?" for f in fields)
        conn.execute(f"UPDATE employees SET {sets} WHERE id=?", vals + [employee_id])
    else:
        conn.execute(f"""
            INSERT INTO employees ({",".join(fields)},created_at)
            VALUES ({",".join(["?"]*len(fields))},?)
        """, vals + [now])
    conn.commit()
    conn.close()

def delete_employee(employee_id):
    conn = db()
    conn.execute("UPDATE employees SET active=0, updated_at=? WHERE id=?",
                 (datetime.now().isoformat(timespec="seconds"), employee_id))
    conn.commit()
    conn.close()

def import_excel(file):
    df = pd.read_excel(file)
    # 支持中文列名 + 英文内部列名
    mapping = {
        "员工编号":"employee_no","员工姓名":"name","部门":"department","护照号":"passport_no",
        "国籍":"nationality","签证类型":"visa_label","Visa Index":"visa_index",
        "许可类型":"permit_type","签证签发日":"visa_issue_date","签证到期日":"visa_expiry_date",
        "停留许可开始日":"stay_permit_start_date","停留许可到期日":"stay_permit_expiry_date",
        "最近入境日":"last_entry_date","已续签次数":"extension_count","Sponsor":"sponsor",
        "负责人":"responsible_person","备注":"notes","employee_no":"employee_no","name":"name",
        "stay_permit_expiry_date":"stay_permit_expiry_date","permit_type":"permit_type"
    }
    df = df.rename(columns=mapping)
    required = ["employee_no","name","visa_label","permit_type","stay_permit_expiry_date"]
    missing = [c for c in required if c not in df.columns]
    if missing:
        raise ValueError("缺少字段：" + ", ".join(missing))
    for c in ["visa_issue_date","visa_expiry_date","stay_permit_start_date","stay_permit_expiry_date","last_entry_date"]:
        if c in df.columns:
            df[c] = pd.to_datetime(df[c], errors="coerce").dt.strftime("%Y-%m-%d")
    for _, r in df.iterrows():
        data = {k: (None if pd.isna(v) else v) for k,v in r.to_dict().items()}
        data.setdefault("nationality","CHN")
        data.setdefault("active",1)
        data.setdefault("extension_count",0)
        add_or_update_employee(data)

def export_excel():
    df = get_employees(active_only=False)
    rows = []
    for _, row in df.iterrows():
        info = calc_info(row)
        rr = row.to_dict()
        rr["剩余天数"] = info.get("days_left")
        rr["状态"] = info.get("status")
        rr["建议最晚办理日"] = info.get("recommended_latest_date")
        rr["法定最晚申请日"] = info.get("legal_latest_date")
        rr["法定最早申请日"] = info.get("legal_earliest_date")
        rows.append(rr)
    out = pd.DataFrame(rows)
    rename = {
        "employee_no":"员工编号","name":"员工姓名","department":"部门","passport_no":"护照号",
        "visa_label":"签证类别","visa_index":"Visa Index","permit_type":"许可类型",
        "visa_issue_date":"签证签发日","visa_expiry_date":"签证到期日",
        "stay_permit_start_date":"停留许可开始日","stay_permit_expiry_date":"停留许可到期日",
        "last_entry_date":"最近入境日","extension_count":"已续签次数","sponsor":"Sponsor",
        "responsible_person":"负责人","notes":"备注","剩余天数":"距离到期天数",
        "状态":"状态","建议最晚办理日":"建议最晚办理日","法定最晚申请日":"法定最晚申请日",
        "法定最早申请日":"法定最早申请日"
    }
    return out.rename(columns=rename)

def main():
    st.set_page_config(page_title="印尼员工签证/停留许可续签提醒", page_icon="🛂", layout="wide")
    init_db()

    st.title("🛂 签证/停留许可续签提醒系统")
    st.caption("适用于在印尼当地机构的人员证件台账管理；日期计算以实际 Izin Tinggal 到期日为核心。")

    df = get_employees(active_only=True)
    today = date.today()

    # Dashboard
    if len(df):
        infos = [calc_info(r) for _, r in df.iterrows()]
        days = [x["days_left"] for x in infos]
        c1,c2,c3,c4 = st.columns(4)
        c1.metric("在册员工", len(df))
        c2.metric("30天内到期", sum(d <= 30 and d >= 0 for d in days))
        c3.metric("7天内到期", sum(d <= 7 and d >= 0 for d in days))
        c4.metric("已过期", sum(d < 0 for d in days))

    tab1, tab2, tab3, tab4, tab5 = st.tabs(["📊 总览/查询","👤 员工档案","📥 批量导入","⚙️ 规则设置","📚 印尼政策口径"])

    with tab1:
        q = st.text_input("搜索员工姓名 / 编号 / 部门 / 护照号", placeholder="例如：张三、EMP001、财务")
        show = df.copy()
        if q:
            mask = (
                show["name"].fillna("").astype(str).str.contains(q, case=False, na=False) |
                show["employee_no"].fillna("").astype(str).str.contains(q, case=False, na=False) |
                show["department"].fillna("").astype(str).str.contains(q, case=False, na=False) |
                show["passport_no"].fillna("").astype(str).str.contains(q, case=False, na=False)
            )
            show = show[mask]
        if len(show):
            view = []
            for _, row in show.iterrows():
                info = calc_info(row)
                view.append({
                    "员工编号": row["employee_no"],
                    "姓名": row["name"],
                    "部门": row["department"],
                    "Visa": row["visa_index"],
                    "许可": row["permit_type"],
                    "停留许可到期": row["stay_permit_expiry_date"],
                    "距离到期(天)": info["days_left"],
                    "建议最晚办理": fmt_date(info["recommended_latest_date"]),
                    "法定最晚申请": fmt_date(info["legal_latest_date"]),
                    "状态": info["status"],
                })
            st.dataframe(pd.DataFrame(view), use_container_width=True, hide_index=True)

            selected = st.selectbox("点击/选择员工查看详细信息",
                                    show["employee_no"].astype(str).tolist())
            row = show[show["employee_no"].astype(str)==str(selected)].iloc[0]
            info = calc_info(row)

            st.subheader(f"员工详情：{row['name']}（{row['employee_no']}）")
            a,b,c,d = st.columns(4)
            a.metric("停留许可到期日", row["stay_permit_expiry_date"])
            b.metric("距离到期", f"{info['days_left']} 天")
            c.metric("建议最晚办理", fmt_date(info["recommended_latest_date"]))
            d.metric("当前状态", info["status"])

            st.write({
                "签证类别": row["visa_label"],
                "Visa Index": row["visa_index"],
                "许可类型": row["permit_type"],
                "签证到期日": row["visa_expiry_date"],
                "停留许可开始日": row["stay_permit_start_date"],
                "最近入境日": row["last_entry_date"],
                "Sponsor": row["sponsor"],
                "负责人": row["responsible_person"],
                "续签次数": row["extension_count"],
                "备注": row["notes"],
            })

            if info["rule"]:
                with st.expander("适用规则"):
                    st.write(info["rule"]["source_note"])

        else:
            st.info("没有匹配员工。")

        st.divider()
        st.download_button(
            "📤 导出当前全部台账（Excel）",
            data=(lambda: export_excel())().to_excel if False else b"",
            disabled=True,
            help="请使用下面的生成按钮。"
        )
        if st.button("生成并下载 Excel"):
            import io
            bio = io.BytesIO()
            export_excel().to_excel(bio, index=False)
            st.download_button("⬇️ 下载台账", data=bio.getvalue(),
                               file_name=f"印尼签证台账_{today}.xlsx",
                               mime="application/vnd.openxmlformats-officedocument.spreadsheetml.sheet")

    with tab2:
        mode = st.radio("操作", ["新增员工","编辑/停用员工"], horizontal=True)
        current = None
        if mode == "编辑/停用员工" and len(df):
            no = st.selectbox("选择员工", df["employee_no"].astype(str).tolist())
            current = df[df["employee_no"].astype(str)==str(no)].iloc[0].to_dict()
            if st.button("停用该员工"):
                delete_employee(int(current["id"]))
                st.success("已停用。刷新页面后生效。")
        elif mode == "编辑/停用员工":
            st.info("暂无员工数据。")
        else:
            current = {
                "employee_no":"","name":"","department":"","passport_no":"","nationality":"CHN",
                "visa_label":"工作签","visa_index":"E23","permit_type":"ITAS",
                "visa_issue_date":"","visa_expiry_date":"","stay_permit_start_date":"",
                "stay_permit_expiry_date":"","last_entry_date":"","extension_count":0,
                "sponsor":"","responsible_person":"","notes":"","active":1
            }

        if current is not None:
            with st.form("employee_form"):
                left,right = st.columns(2)
                with left:
                    employee_no = st.text_input("员工编号*", value=str(current.get("employee_no") or ""))
                    name = st.text_input("员工姓名*", value=str(current.get("name") or ""))
                    department = st.text_input("部门", value=str(current.get("department") or ""))
                    passport_no = st.text_input("护照号", value=str(current.get("passport_no") or ""))
                    nationality = st.text_input("国籍", value=str(current.get("nationality") or "CHN"))
                    visa_label = st.text_input("签证类别（内部名称）", value=str(current.get("visa_label") or ""))
                    visa_codes = pd.read_sql_query(
                        "SELECT code FROM rules ORDER BY visa_family, code", db()
                    )["code"].astype(str).tolist()
                    current_code = str(current.get("visa_index") or "")
                    if current_code and current_code not in visa_codes:
                        visa_codes.append(current_code)
                    if "其他" not in visa_codes:
                        visa_codes.append("其他")
                    visa_index = st.selectbox(
                        "Visa Index",
                        visa_codes,
                        index=visa_codes.index(current_code) if current_code in visa_codes else len(visa_codes)-1
                    )
                    permit_type = st.selectbox("实际停留许可类型",
                                               ["ITK","ITAS"],
                                               index=0 if str(current.get("permit_type"))!="ITAS" else 1)
                    sponsor = st.text_input("Sponsor/担保人", value=str(current.get("sponsor") or ""))
                    responsible_person = st.text_input("证件负责人", value=str(current.get("responsible_person") or ""))
                with right:
                    def dinput(label,key):
                        val = parse_date(current.get(key))
                        return st.date_input(label, value=val or date.today(), key=f"{key}_x")
                    visa_issue_date = dinput("签证签发日", "visa_issue_date")
                    visa_expiry_date = dinput("签证到期日", "visa_expiry_date")
                    stay_permit_start_date = dinput("停留许可开始日*", "stay_permit_start_date")
                    stay_permit_expiry_date = dinput("停留许可到期日*", "stay_permit_expiry_date")
                    last_entry_date = dinput("最近入境日", "last_entry_date")
                    extension_count = st.number_input("已续签次数", min_value=0, step=1,
                                                      value=int(current.get("extension_count") or 0))
                    notes = st.text_area("备注", value=str(current.get("notes") or ""))

                submitted = st.form_submit_button("保存")
                if submitted:
                    if not employee_no or not name or not stay_permit_expiry_date:
                        st.error("员工编号、姓名、停留许可到期日必须填写。")
                    else:
                        data = {
                            "employee_no":employee_no,"name":name,"department":department,
                            "passport_no":passport_no,"nationality":nationality,
                            "visa_label":visa_label,"visa_index":visa_index,
                            "permit_type":permit_type,
                            "visa_issue_date":fmt_date(visa_issue_date),
                            "visa_expiry_date":fmt_date(visa_expiry_date),
                            "stay_permit_start_date":fmt_date(stay_permit_start_date),
                            "stay_permit_expiry_date":fmt_date(stay_permit_expiry_date),
                            "last_entry_date":fmt_date(last_entry_date),
                            "extension_count":extension_count,"sponsor":sponsor,
                            "responsible_person":responsible_person,"notes":notes,"active":1
                        }
                        add_or_update_employee(data, int(current["id"]) if current.get("id") else None)
                        st.success("保存成功。")

    with tab3:
        st.markdown("上传 Excel 后批量导入。必填列：`员工编号、员工姓名、签证类型、许可类型、停留许可到期日`。")
        template = pd.DataFrame([{
            "员工编号":"EMP001","员工姓名":"张三","部门":"项目部","护照号":"E12345678",
            "国籍":"CHN","签证类型":"工作签","Visa Index":"E23","许可类型":"ITAS",
            "签证签发日":"2026-01-01","签证到期日":"2026-12-31",
            "停留许可开始日":"2026-01-10","停留许可到期日":"2027-01-09",
            "最近入境日":"2026-01-10","已续签次数":0,"Sponsor":"XXX PT",
            "负责人":"HR","备注":""
        }])
        import io
        bio = io.BytesIO(); template.to_excel(bio,index=False)
        st.download_button("下载 Excel 模板", data=bio.getvalue(),
                           file_name="印尼签证台账模板.xlsx",
                           mime="application/vnd.openxmlformats-officedocument.spreadsheetml.sheet")
        uploaded = st.file_uploader("上传 Excel", type=["xlsx"])
        if uploaded and st.button("开始导入"):
            try:
                import_excel(uploaded)
                st.success("导入完成。重复员工编号会更新。")
            except Exception as e:
                st.error(str(e))

    with tab4:
        st.subheader("⚙️ 规则设置")

        st.markdown("### A. 到期状态判定规则")
        st.caption(
            "只能修改状态名称、最小天数、最大天数和说明；不能新增或删除。"
        )

        status_df = pd.DataFrame(
            get_status_rules(),
            columns=["id","status_name","min_days","max_days","display_order","description"]
        ).drop(columns=["id"])

        edited_status = st.data_editor(
            status_df,
            use_container_width=True,
            hide_index=True,
            num_rows="fixed",
            disabled=["display_order"],
            column_config={
                "status_name": st.column_config.TextColumn("状态名称", required=True),
                "min_days": st.column_config.NumberColumn("最小天数", help="留空表示无下限"),
                "max_days": st.column_config.NumberColumn("最大天数", help="留空表示无上限"),
                "display_order": st.column_config.NumberColumn("排序", disabled=True),
                "description": st.column_config.TextColumn("说明"),
            },
            key="fixed_status_rules_editor",
        )

        if st.button("保存状态判定规则", type="primary"):
            try:
                save_status_rules(edited_status.to_dict("records"))
                st.success("状态判定规则已保存。")
                st.rerun()
            except Exception as e:
                st.error(f"保存失败：{e}")

        st.divider()

        st.markdown("### B. 签证类别 / Visa Index 政策规则")
        st.caption(
            "允许新增、删除、修改签证类别。调整后，请点击“保存签证类别规则”。"
        )

        policy_df = pd.read_sql_query("""
            SELECT code, visa_family, description,
                   earliest_start_days, legal_latest_before_days,
                   recommended_buffer_days, max_total_stay_days,
                   extension_count, source_note
            FROM rules
            ORDER BY visa_family, code
        """, db())

        edited_policy = st.data_editor(
            policy_df,
            use_container_width=True,
            hide_index=True,
            num_rows="dynamic",
            column_config={
                "code": st.column_config.TextColumn("Visa Index / 规则代码", required=True),
                "visa_family": st.column_config.TextColumn("签证类别 / Family", required=True),
                "description": st.column_config.TextColumn("用途说明"),
                "earliest_start_days": st.column_config.NumberColumn("最早申请窗口(天)", min_value=0),
                "legal_latest_before_days": st.column_config.NumberColumn("法定最晚提前天数", min_value=0),
                "recommended_buffer_days": st.column_config.NumberColumn("内部建议缓冲(天)", min_value=0),
                "max_total_stay_days": st.column_config.NumberColumn("最长累计停留天数", min_value=0),
                "extension_count": st.column_config.NumberColumn("规则续签次数", min_value=0),
                "source_note": st.column_config.TextColumn("政策来源/备注"),
            },
            key="visa_policy_rules_editor",
        )

        if st.button("保存签证类别规则", type="primary"):
            try:
                save_policy_rules(edited_policy.to_dict("records"))
                st.success("签证类别规则已保存。")
                st.rerun()
            except Exception as e:
                st.error(f"保存失败：{e}")


    with tab5:
        st.subheader("印尼政策口径（程序采用的核心规则）")
        st.markdown("""
**1. 访问类/旅游/商务：ITK（Izin Tinggal Kunjungan）**

本系统不把贴签页上的 Visa Valid Until 直接当作员工在印尼的合法停留终点。
程序重点使用员工实际的 **Izin Tinggal 到期日**。对于访问类签证，具体 Visa Index
（例如 C1/C2、B1/B2 等）必须与电子签证/入境记录核对。

**2. VoA 类 B1 等**

官方移民局页面显示：B1 的首次停留最多 30 天，可延长一次，累计最多 60 天。
因此系统会在到期前触发提醒；“建议最晚办理日”是内部风险控制日期，不表述为法律强制期限。

**3. ITAS 工作签**

官方移民局页面：期限 **不超过1年的 ITAS**，续签申请最早可在到期前30天提出，
最晚可在原 ITAS 到期当天申请；期限 **超过1年的 ITAS**，最早可提前3个月申请，
最晚同样到期当天。若申请并缴费已在 ITAS 到期前完成，即便审批跨过原到期日，
官方说明不作为 overstay 处理。

**4. 本程序的推荐日期**

“建议最晚办理日”不是法律规定，而是内部合规缓冲：
- ITAS ≤ 1年：默认提前45天作为提醒节点；
- ITAS > 1年：默认提前90天；
- ITK/VoA：默认提前14～30天（按规则表），并可由管理员调整。

正式办理仍应以印尼移民局（Direktorat Jenderal Imigrasi）、eVisa/相应 kantor imigrasi
以及员工实际签发的 e-ITAS/e-ITK 为准。
        """)
        st.info("建议HR每天打开本系统，查看“30天内到期”“7天内到期”“已过期”三个指标，并由负责人对异常项进行人工复核。")

if __name__ == "__main__":
    main()
