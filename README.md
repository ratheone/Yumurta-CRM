# -*- coding: utf-8 -*-
"""
streamlit run yumurta.py
Muhammed KOÇ
"""

import streamlit as st
import sqlite3
import pandas as pd
from datetime import datetime, date, timedelta
import time
import hashlib

#Ayarlar şifre & data
DB_FILE = 'yumurta.db'
DEFAULT_PASS_HASH = "8d969eef6ecad3c29a3a629280e686cf0c3f5d5a86aff3ca12020c923adc6c92" 
#şifre görünmemesi için yapay zekadan destek alınmıştır başka bir kullanıcı tarafından okunmaması adına 1234561 olan şifre maskelenmişitir.

DEFAULT_SOURCES = {
    "Tavuk Yumurtası": "S (Küçük - <53g), M (Orta - 53-63g), L (Büyük - 63-73g), XL (Çok Büyük - >73g)",
    "Bıldırcın Yumurtası": "Standart (10-12g)",
    "Ördek Yumurtası": "Standart (70g+)",
    "Kaz Yumurtası": "Standart (140g+)",
    "Hindi Yumurtası": "Standart (70-90g)",
    "Devekuşu Yumurtası": "1 Adet (~1.5 kg)"
}

PRODUCTION_TYPES_CHICKEN = ["Organik (0)", "Serbest Gezen (1)", "Kümes (2)", "Kafes (3)"]
PRODUCTION_TYPES_OTHER = ["Doğal/Standart Üretim"]
NUTRITION_TYPES = ["Standart", "Omega-3 (Kalp/Beyin)", "D Vitamini Zenginleştirilmiş", "Selenyumlu (Antioksidan)", "Probiyotikli (Sindirim)"]
SHELL_COLORS = ["Beyaz", "Kahverengi", "Mavi/Yeşil (Araucana)", "Karışık"]
SUBSCRIPTION_TYPES = ["Haftalık", "Aylık", "Yıllık"]
DAYS_TR_MAP = {0: "Pazartesi", 1: "Salı", 2: "Çarşamba", 3: "Perşembe", 4: "Cuma", 5: "Cumartesi", 6: "Pazar"}

def format_date_tr(date_obj):
    if isinstance(date_obj, str):
        try: date_obj = datetime.strptime(date_obj, "%Y-%m-%d").date()
        except: return date_obj
    if isinstance(date_obj, date) or isinstance(date_obj, datetime):
        day_name = DAYS_TR_MAP[date_obj.weekday()]
        return f"{date_obj.strftime('%d/%m/%Y')} {day_name}"
    return str(date_obj)

def make_hash(password): return hashlib.sha256(str(password).encode('utf-8')).hexdigest()
def check_password_match(input_password, stored_hash): return make_hash(input_password) == stored_hash

#VERİTABANI
def get_db_connection(): return sqlite3.connect(DB_FILE)

def init_db():
    baglanti = get_db_connection()
    c = baglanti.cursor()
    c.execute('''CREATE TABLE IF NOT EXISTS settings (key TEXT PRIMARY KEY, value TEXT)''')
    c.execute("INSERT OR IGNORE INTO settings (key, value) VALUES ('tax_rate', '8.0')")
    c.execute("INSERT OR IGNORE INTO settings (key, value) VALUES ('admin_pass', ?)", (DEFAULT_PASS_HASH,))
    c.execute('''CREATE TABLE IF NOT EXISTS definitions (id INTEGER PRIMARY KEY AUTOINCREMENT, source_name TEXT UNIQUE, sizes TEXT)''')
    check = c.execute("SELECT COUNT(*) FROM definitions").fetchone()[0]
    if check == 0:
        for src, sizes in DEFAULT_SOURCES.items(): c.execute("INSERT INTO definitions (source_name, sizes) VALUES (?, ?)", (src, sizes))
    c.execute('''CREATE TABLE IF NOT EXISTS products (id INTEGER PRIMARY KEY AUTOINCREMENT, source TEXT, production_type TEXT, size TEXT, nutrition TEXT, shell_color TEXT, price REAL, description TEXT)''')
    c.execute('''CREATE TABLE IF NOT EXISTS members (id INTEGER PRIMARY KEY AUTOINCREMENT, name TEXT, phone TEXT, sub_type TEXT, product_id INTEGER, quantity INTEGER, delivery_day TEXT, delivery_time TEXT, start_date DATE, registration_date DATE, FOREIGN KEY (product_id) REFERENCES products (id))''')
    try: c.execute("SELECT start_date FROM members LIMIT 1")
    except sqlite3.OperationalError: c.execute("ALTER TABLE members ADD COLUMN start_date DATE")
    try: c.execute("SELECT registration_date FROM members LIMIT 1")
    except sqlite3.OperationalError: 
        c.execute("ALTER TABLE members ADD COLUMN registration_date DATE")
        c.execute("UPDATE members SET registration_date = start_date WHERE registration_date IS NULL")
    c.execute('''CREATE TABLE IF NOT EXISTS delivery_logs (id INTEGER PRIMARY KEY AUTOINCREMENT, member_id INTEGER, delivery_date DATE, status INTEGER DEFAULT 1, timestamp DATETIME, FOREIGN KEY (member_id) REFERENCES members (id))''')
    baglanti.commit(); baglanti.close()

def get_dynamic_sources():
    baglanti = get_db_connection()
    satirlar = baglanti.execute("SELECT source_name, sizes FROM definitions").fetchall()
    baglanti.close()
    dynamic_dict = {}
    for satir in satirlar: dynamic_dict[satir[0]] = [s.strip() for s in satir[1].split(',')]
    return dynamic_dict

def update_delivery_status(member_id, delivery_date, status_code):
    baglanti = get_db_connection()
    baglanti.execute("DELETE FROM delivery_logs WHERE member_id=? AND delivery_date=?", (member_id, delivery_date))
    if status_code != 0:
        baglanti.execute("INSERT INTO delivery_logs (member_id, delivery_date, status, timestamp) VALUES (?, ?, ?, ?)", (member_id, delivery_date, status_code, datetime.now()))
    baglanti.commit(); baglanti.close()

#ARAYÜZ
st.set_page_config(page_title="Lucy Tavuk CRM", page_icon="🐔", layout="wide")
init_db()
SOURCE_TO_SIZES = get_dynamic_sources()
if 'sepet' not in st.session_state: st.session_state['sepet'] = []

st.markdown("""
<style>
@keyframes pulse-red {
    0% { box-shadow: 0 0 0 0 rgba(255, 0, 0, 0.7); border-color: #ff0000; }
    70% { box-shadow: 0 0 0 10px rgba(255, 0, 0, 0); border-color: #ffcccc; }
    100% { box-shadow: 0 0 0 0 rgba(255, 0, 0, 0); border-color: #ff0000; }
}
.urgent-box {
    animation: pulse-red 2s infinite;
    border: 3px solid red;
    border-radius: 10px;
    padding: 10px;
    margin-bottom: 10px;
    background-color: rgba(255, 0, 0, 0.05);
}
.urgent-text { color: red; font-weight: bold; font-size: 1.1em; }
</style>
""", unsafe_allow_html=True)



#menü aktif olduğunu hafızada tutar
if 'aktif_menu' not in st.session_state:
    st.session_state['aktif_menu'] = "Ana Sayfa"

#Butona basılınca rekli olarak kalacak şekilde çalışacak fonksiyon
def menu_sec(secim):
    st.session_state['aktif_menu'] = secim

#Sidebar 
with st.sidebar:
    st.title("Lucy Yumurta CRM 🐔")
    
    st.divider()
    
#Menüler 
    menu_options = ["Ana Sayfa", "Üyelik Ekle", "Üyeler & Düzenle", "Tür & Fiyat Yönetimi", "Ayarlar"]
    
    for option in menu_options:
        btn_type = "primary" if st.session_state['aktif_menu'] == option else "secondary"
        
        if st.button(option, key=f"nav_{option}", type=btn_type, use_container_width=True):
            menu_sec(option)
            st.rerun()

aktif_menu = st.session_state['aktif_menu']

baglanti = get_db_connection()
try: tax_rate = float(baglanti.execute("SELECT value FROM settings WHERE key='tax_rate'").fetchone()[0])
except: tax_rate = 8.0
admin_pass_hash = baglanti.execute("SELECT value FROM settings WHERE key='admin_pass'").fetchone()[0]
baglanti.close()


#ANA SAYFA
if aktif_menu == "Ana Sayfa":
    bugun = datetime.now().date()
    today_formatted = format_date_tr(bugun)
    bugun_gun_adi = DAYS_TR_MAP[bugun.weekday()]
    simdi_saat = datetime.now().time()
    
    st.title(f"📊 Yönetim Paneli - {today_formatted}")
    
    baglanti = get_db_connection()
    
#Gecikmiş Teslimatlar
    geciken_teslimatlar = []
    for i in range(1, 7):
        check_date = bugun - timedelta(days=i)
        check_day_name = DAYS_TR_MAP[check_date.weekday()]
        sorgu = """
            SELECT m.id, m.name, m.phone, m.quantity, p.description, p.price, m.sub_type, m.delivery_time, m.start_date
            FROM members m 
            JOIN products p ON m.product_id = p.id
            WHERE m.delivery_day = ? 
            AND m.start_date <= ?
            AND m.id NOT IN (SELECT member_id FROM delivery_logs WHERE delivery_date = ?)
        """
        satirlar = baglanti.execute(sorgu, (check_day_name, check_date, check_date)).fetchall()
        for r in satirlar:
            kayit = dict(zip(['id', 'name', 'phone', 'quantity', 'description', 'price', 'sub_type', 'delivery_time', 'start_date'], r))
            kayit['missed_date'] = check_date
            geciken_teslimatlar.append(kayit)

    if geciken_teslimatlar:
        st.warning(f"⚠️ DİKKAT: Son 7 gün içinde işlemi yapılmamış {len(geciken_teslimatlar)} teslimat var!")
        with st.expander("Gecikmiş Teslimatları Görüntüle / İşlem Yap", expanded=True):
            for kayit in geciken_teslimatlar:
                m_date_str = format_date_tr(kayit['missed_date'])
                with st.container():
                    c1, c2, c3, c4 = st.columns([2, 3, 2, 2])
                    c1.error(f"📅 {m_date_str}")
                    c2.write(f"**{kayit['name']}**\n{kayit['phone']}")
                    c3.write(f"{kayit['quantity']} Adet\n{kayit['description'].split(' - ')[0]}")
                    with c4:
                        if st.button("Teslim", key=f"miss_ok_{kayit['id']}_{kayit['missed_date']}"):
                            update_delivery_status(kayit['id'], kayit['missed_date'], 1); st.rerun()
                        if st.button("İptal", key=f"miss_cancel_{kayit['id']}_{kayit['missed_date']}"):
                            update_delivery_status(kayit['id'], kayit['missed_date'], 2); st.rerun()
                st.divider()


    df_members = pd.read_sql_query("""
        SELECT m.id, m.name, m.phone, m.quantity, p.description, p.price, m.sub_type, m.delivery_time, m.start_date
        FROM members m JOIN products p ON m.product_id = p.id WHERE m.delivery_day = ? AND m.start_date <= ?
    """, baglanti, params=(bugun_gun_adi, bugun))
    
    log_rows = baglanti.execute("SELECT member_id, status FROM delivery_logs WHERE delivery_date=?", (bugun,)).fetchall()
    durum_haritasi = {row[0]: row[1] for row in log_rows}
    
    cancelled_data = pd.read_sql_query("""
        SELECT l.delivery_date, m.quantity, p.price
        FROM delivery_logs l JOIN members m ON l.member_id = m.id JOIN products p ON m.product_id = p.id WHERE l.status = 2
    """, baglanti)
    
    current_month = datetime.now().month
    current_year = datetime.now().year
    aylik_kayip = 0.0
    yillik_kayip = 0.0
    
    if not cancelled_data.empty:
        cancelled_data['date_obj'] = pd.to_datetime(cancelled_data['delivery_date'])
        for _, satir in cancelled_data.iterrows():
            loss = satir['quantity'] * satir['price']
            if satir['date_obj'].year == current_year:
                yillik_kayip += loss
                if satir['date_obj'].month == current_month: aylik_kayip += loss

    total_active_members = baglanti.execute("SELECT COUNT(DISTINCT phone) FROM members").fetchone()[0]
    
    gunluk_ciro = 0
    if not df_members.empty:
        for _, satir in df_members.iterrows():
            m_id = satir['id']
            status = durum_haritasi.get(m_id, 0)
            if status != 2: gunluk_ciro += (satir['quantity'] * satir['price'])

    df_all = pd.read_sql_query("SELECT m.sub_type, m.quantity, p.price FROM members m JOIN products p ON m.product_id = p.id", baglanti)
    potansiyel_aylik = 0
    if not df_all.empty:
        for _, satir in df_all.iterrows():
            rev = satir['quantity'] * satir['price']
            if satir['sub_type'] == "Haftalık": potansiyel_aylik += (rev * 4)
            elif satir['sub_type'] == "Aylık": potansiyel_aylik += rev
            elif satir['sub_type'] == "Yıllık": potansiyel_aylik += (rev / 12)
    
    final_monthly_revenue = potansiyel_aylik - aylik_kayip
    net_yearly_profit = (potansiyel_aylik * 12 - yillik_kayip) * (1 - (tax_rate/100))
    
    col1, col2, col3, col4 = st.columns(4)
    col1.metric("Aktif Müşteri", f"{total_active_members}")
    col2.metric("Günlük Ciro (Net)", f"₺{gunluk_ciro:,.2f}")
    col3.metric("Aylık Ciro", f"₺{final_monthly_revenue:,.2f}", delta=f"-₺{aylik_kayip:,.2f} Kayıp")
    col4.metric(f"Yıllık Net Kar", f"₺{net_yearly_profit:,.2f}")
    
    st.divider()
    st.subheader(f"🚚 Teslimat Listesi ({len(df_members)} Sipariş)")
    
    if df_members.empty:
        st.info("Bugün için planlanmış ve (tarihi gelmiş) teslimat bulunmamaktadır.")
    else:
        c1, c2, c3, c4, c5 = st.columns([2.5, 2, 1, 1, 2])
        c1.markdown("**Müşteri & Abonelik**")
        c2.markdown("**Ürün & Adet ℹ️**")
        c3.markdown("**Saat**")
        c4.markdown("**Tutar**")
        c5.markdown("**İşlem**")
        st.markdown("---")
        
        for index, satir in df_members.iterrows():
            m_id = satir['id']
            status = durum_haritasi.get(m_id, 0) 
            total_price = satir['quantity'] * satir['price']
            
            try:
                start_dt = datetime.strptime(satir['start_date'], "%Y-%m-%d").date()
                end_dt = start_dt.replace(year=start_dt.year + 1)
                if bugun > end_dt: continue
                date_info = f"📅 Baş: {start_dt.strftime('%d/%m/%Y')} | Bit: {end_dt.strftime('%d/%m/%Y')}"
            except: date_info = ""

            tooltip_text = f"📦 Ürün: {satir['description']}\n💰 Birim: {satir['price']} ₺\n🔄 Abonelik: {satir['sub_type']}"
            
            try: delivery_time_obj = datetime.strptime(str(satir['delivery_time'])[:5], "%H:%M").time()
            except: delivery_time_obj = datetime.strptime("09:00", "%H:%M").time()
            is_urgent = (status == 0) and (simdi_saat >= delivery_time_obj)

            if status == 1:
                with st.success(f"✅ Teslim Edildi - {satir['name']}"):
                    col_name, col_prod, col_time, col_price, col_action = st.columns([2.5, 2, 1, 1, 2])
                    with col_name: st.write(f"**{satir['name']}**"); st.caption(date_info)
                    with col_prod: st.markdown(f"**{satir['quantity']} Adet**", help=tooltip_text)
                    with col_time: st.text("Ok")
                    with col_price: st.write(f"**₺{total_price:.2f}**")
                    with col_action:
                        with st.expander("🔒 Geri Al"):
                            p_in = st.text_input("Şifre", type="password", key=f"p1_{m_id}")
                            if st.button("Onayla", key=f"b1_{m_id}"):
                                if check_password_match(p_in, admin_pass_hash): update_delivery_status(m_id, bugun, 0); st.rerun()
                                else: st.error("Hatalı")
            elif status == 2:
                with st.error(f"❌ İptal Edildi - {satir['name']}"):
                    col_name, col_prod, col_time, col_price, col_action = st.columns([2.5, 2, 1, 1, 2])
                    with col_name: st.markdown(f"~~{satir['name']}~~"); st.caption(date_info)
                    with col_prod: st.markdown(f"~~{satir['quantity']} Adet~~", help=tooltip_text)
                    with col_time: st.caption("-")
                    with col_price: st.markdown(f"**- ₺{total_price:.2f}**")
                    with col_action:
                        with st.expander("🔒 Geri Al"):
                            p_in = st.text_input("Şifre", type="password", key=f"p2_{m_id}")
                            if st.button("Onayla", key=f"b2_{m_id}"):
                                if check_password_match(p_in, admin_pass_hash): update_delivery_status(m_id, bugun, 0); st.rerun()
                                else: st.error("Hatalı")
            else: #zamanı geldi geçti kısmını yapay zekadan destek alınarak uyarlama yapılmışitırçü.
                if is_urgent: st.markdown(f'<div class="urgent-box"><div class="urgent-text">⏰ ZAMANI GELDİ / GECİKTİ ({str(delivery_time_obj)[:5]})</div>', unsafe_allow_html=True)
                with st.container():
                    col_name, col_prod, col_time, col_price, col_action = st.columns([2.5, 2, 1, 1, 2])
                    with col_name: st.write(f"**{satir['name']}**"); st.caption(f"📞 {satir['phone']}"); st.caption(date_info)
                    with col_prod: st.markdown(f"**{satir['quantity']} Adet**\n{satir['description'].split(' - ')[0]}", help=tooltip_text)
                    with col_time: st.text(str(satir['delivery_time'])[:5])
                    with col_price: st.write(f"**₺{total_price:.2f}**")
                    with col_action:
                        confirm_key = f"confirm_action_{m_id}"
                        if st.session_state.get(confirm_key):
                            st.caption("Emin misiniz?")
                            cy, cn = st.columns(2)
                            if cy.button("Evet", key=f"y_{m_id}"):
                                new_s = 1 if st.session_state[confirm_key] == "deliver" else 2
                                update_delivery_status(m_id, bugun, new_s); del st.session_state[confirm_key]; st.rerun()
                            if cn.button("Hayır", key=f"n_{m_id}"): del st.session_state[confirm_key]; st.rerun()
                        else:
                            b1, b2 = st.columns(2)
                            if b1.button("Teslim", key=f"pd_{m_id}", type="primary"): st.session_state[confirm_key] = "deliver"; st.rerun()
                            if b2.button("İptal", key=f"pc_{m_id}", type="secondary"): st.session_state[confirm_key] = "cancel"; st.rerun()
                if is_urgent: st.markdown('</div>', unsafe_allow_html=True)
                st.markdown("---")
    baglanti.close()


#YUMURTA TÜRLERİ
elif aktif_menu == "Tür & Fiyat Yönetimi":
    st.title("🥚 Ürün Kataloğu ve Tanımlamar")
    with st.expander("🐣 LİSTEDE OLMAYAN YENİ BİR TÜR EKLE", expanded=False):
        c_new1, c_new2 = st.columns(2)
        with c_new1: new_def_name = st.text_input("Yeni Yumurta Türü Adı", placeholder="Örn: Keklik Yumurtası")
        with c_new2: new_def_sizes = st.text_input("Boyutlar", placeholder="Standart, Büyük")
        if st.button("Yeni Türü Sisteme Ekle"):
            if new_def_name and new_def_sizes:
                try:
                    baglanti = get_db_connection()
                    baglanti.execute("INSERT INTO definitions (source_name, sizes) VALUES (?, ?)", (new_def_name, new_def_sizes))
                    baglanti.commit(); baglanti.close()
                    st.success(f"{new_def_name} eklendi!"); time.sleep(1.5); st.rerun()
                except sqlite3.IntegrityError: st.error("Kayıtlı!")

    st.markdown("---")
    st.subheader("➕ Yeni Satış Ürünü Fiyatlandır")
    c1, c2 = st.columns(2)
    with c1:
        s_src = st.selectbox("Kaynak", list(SOURCE_TO_SIZES.keys()))
        s_size = st.selectbox("Boyut", SOURCE_TO_SIZES[s_src])
        s_prod = st.selectbox("Üretim", PRODUCTION_TYPES_CHICKEN if "Tavuk" in s_src else PRODUCTION_TYPES_OTHER)
    with c2:
        s_nut = st.selectbox("Besin", NUTRITION_TYPES)
        s_shell = st.selectbox("Kabuk", SHELL_COLORS)
        s_price = st.number_input("Fiyat", min_value=0.0, step=0.25)
        gen_name = f"{s_src} - {s_prod.split('(')[0].strip()} - {s_size.split('(')[0].strip()} [{s_nut}]"
        st.text_input("İsim", value=gen_name, disabled=True)

    if st.button("✅ Kaydet"):
        baglanti = get_db_connection()
        baglanti.execute("INSERT INTO products (source, production_type, size, nutrition, shell_color, price, description) VALUES (?,?,?,?,?,?,?)",
                     (s_src, s_prod, s_size, s_nut, s_shell, s_price, gen_name))
        baglanti.commit(); baglanti.close(); st.success("Eklendi!"); time.sleep(1); st.rerun()

    st.divider(); st.subheader("📋 Mevcut Fiyat Listesi")
    baglanti = get_db_connection()
    df_p = pd.read_sql_query("SELECT * FROM products", baglanti)
    baglanti.close()
    
    if not df_p.empty:
        ALL_S = []
        for v in SOURCE_TO_SIZES.values(): ALL_S.extend(v)
        ALL_S = list(set(ALL_S))
        col_conf = {
            "id": st.column_config.NumberColumn("ID", disabled=True),
            "source": st.column_config.SelectboxColumn("Kaynak", options=list(SOURCE_TO_SIZES.keys()), required=True),
            "production_type": st.column_config.SelectboxColumn("Üretim", options=PRODUCTION_TYPES_CHICKEN + PRODUCTION_TYPES_OTHER, required=True),
            "size": st.column_config.SelectboxColumn("Boyut", options=ALL_S, required=True),
            "nutrition": st.column_config.SelectboxColumn("Besin", options=NUTRITION_TYPES, required=True),
            "shell_color": st.column_config.SelectboxColumn("Kabuk", options=SHELL_COLORS, required=True),
            "price": st.column_config.NumberColumn("Fiyat", format="%.2f"),
            "description": st.column_config.TextColumn("Açıklama", disabled=True)
        }
        ed_df = st.data_editor(df_p, column_config=col_conf, hide_index=True, use_container_width=True, num_rows="fixed", key="p_edit")
        if st.button("💾 Kaydet"):
            baglanti = get_db_connection()
            for i, r in ed_df.iterrows():
                nd = f"{r['source']} - {r['production_type'].split('(')[0].strip()} - {r['size'].split('(')[0].strip()} [{r['nutrition']}]"
                baglanti.execute("UPDATE products SET source=?, production_type=?, size=?, nutrition=?, shell_color=?, price=?, description=? WHERE id=?",
                             (r['source'], r['production_type'], r['size'], r['nutrition'], r['shell_color'], r['price'], nd, r['id']))
            baglanti.commit(); baglanti.close(); st.success("Güncellendi!"); time.sleep(1); st.rerun()

        st.markdown("🗑️ Çoklu Silme")
        sel_del = st.multiselect("Silinecekleri Seç", [f"{r['id']} - {r['description']}" for i, r in df_p.iterrows()])
        if sel_del:
            
#ŞİFRE KORUMASI EKLENDİ destek alındı
            del_pass_p = st.text_input("Yönetici Şifresi", type="password", key="prod_del_pass")
            if st.button("🗑️ Sil", type="secondary"):
                if check_password_match(del_pass_p, admin_pass_hash):
                    ids = [(int(x.split(' - ')[0]),) for x in sel_del]
                    baglanti = get_db_connection()
                    baglanti.executemany("DELETE FROM products WHERE id=?", ids)
                    baglanti.commit(); baglanti.close(); st.warning("Silindi."); time.sleep(1); st.rerun()
                else: st.error("Hatalı Şifre")
    else: st.info("Boş")


#ÜYELİK EKLEMWE ALANI
elif aktif_menu == "Üyelik Ekle":
    st.title("📝 Abonelik Oluştur")
    baglanti = get_db_connection()
    df_prod = pd.read_sql_query("SELECT * FROM products", baglanti)
    baglanti.close()
    
    if df_prod.empty:
        st.error("Önce ürün ekleyin.")
    else:
        c1, c2 = st.columns(2)
        with c1:

            isim = st.text_input("Ad Soyad", placeholder="Müşteri ismi")
            tel = st.text_input("Telefon", placeholder="+90 5xx xxx xx xx")
            sub_type = st.selectbox("Abonelik", SUBSCRIPTION_TYPES)
            ilk_gun = st.date_input("İlk Teslimat Başlangıç Tarihi", value=datetime.now())
            saat = st.time_input("Saat", value=datetime.strptime("09:00", "%H:%M").time())
            

            gun_adi_str = DAYS_TR_MAP[ilk_gun.weekday()]
            st.info(f"Kayıt: {format_date_tr(datetime.now())}\nİlk Teslimat: **{format_date_tr(ilk_gun)}**\nGün: **{gun_adi_str}**")

        with c2:
            src = st.selectbox("Kaynak", df_prod['source'].unique())
            df2 = df_prod[df_prod['source'] == src]
            prod = st.selectbox("Üretim", df2['production_type'].unique())
            df3 = df2[df2['production_type'] == prod]
            size = st.selectbox("Boyut", df3['size'].unique())
            df_fin = df3[df3['size'] == size]
            vars = {f"{r['nutrition']} | {r['shell_color']} | {r['price']}₺": r['id'] for _, r in df_fin.iterrows()}
            det = st.selectbox("Detay", list(vars.keys()))
            pid = vars[det]
            adet = st.number_input("Adet", 1, value=30)
            
            if st.button("➕ Ekle"):
                pr = df_fin[df_fin['id'] == pid]['price'].values[0]
                st.session_state['sepet'].append({"pid": int(pid), "desc": f"{src}, {prod}, {size} ({det})", "adet": adet, "birim_fiyat": pr, "toplam": pr*adet})
                st.success("Sepette")

        if st.session_state['sepet']:
            st.dataframe(pd.DataFrame(st.session_state['sepet']), use_container_width=True)
            if st.button("✅ Kaydet", type="primary"):
                if isim and tel:
                    baglanti = get_db_connection()
                    for i in st.session_state['sepet']:
                        baglanti.execute("INSERT INTO members (name, phone, sub_type, product_id, quantity, delivery_day, delivery_time, start_date, registration_date) VALUES (?,?,?,?,?,?,?,?,?)",
                                     (isim, tel, sub_type, i['pid'], i['adet'], DAYS_TR_MAP[ilk_gun.weekday()], str(saat), ilk_gun, datetime.now().date()))
                    baglanti.commit(); baglanti.close(); st.session_state['sepet'] = []; st.success("Kaydedildi!"); time.sleep(2); st.rerun()
                else: st.error("Bilgileri giriniz.")


#ÜYELER & DÜZENLEME
elif aktif_menu == "Üyeler & Düzenle":
    st.title("👥 Üyeler")
    baglanti = get_db_connection()
    df_m = pd.read_sql_query("""
        SELECT m.id, m.name, m.phone, m.sub_type, p.description, m.quantity, m.delivery_day, m.delivery_time, m.start_date, m.registration_date 
        FROM members m JOIN products p ON m.product_id = p.id ORDER BY m.id DESC
    """, baglanti)
    baglanti.close()
    
    if not df_m.empty:
        df_m['fmt_reg_date'] = df_m['registration_date'].apply(format_date_tr)
        df_m['fmt_start_date'] = df_m['start_date'].apply(format_date_tr)
        st.dataframe(df_m[['id', 'name', 'phone', 'sub_type', 'description', 'quantity', 'fmt_reg_date', 'fmt_start_date']], use_container_width=True, column_config={"fmt_reg_date": "Kayıt", "fmt_start_date": "İlk Teslimat"})
    
    with st.expander("✏️ Üye Bilgilerini Güncelle", expanded=False):
        if not df_m.empty:
            member_dict = {f"{r['id']} - {r['name']}": r['id'] for _, r in df_m.iterrows()}
            selected_m = st.selectbox("Düzenlenecek Üyeyi Seçin", list(member_dict.keys()))
            m_id = member_dict[selected_m]
            baglanti = get_db_connection()
            curr = baglanti.execute("SELECT * FROM members WHERE id=?", (m_id,)).fetchone()
            baglanti.close()
            if curr:
                with st.form("edit_mem"):
                    c_up1, c_up2 = st.columns(2)
                    with c_up1:

                        guncel_isim = st.text_input("Ad Soyad", value=curr[1])
                        guncel_tel = st.text_input("Telefon", value=curr[2])
                        try: s_idx = SUBSCRIPTION_TYPES.index(curr[3])
                        except: s_idx = 0
                        up_sub = st.selectbox("Abonelik", SUBSCRIPTION_TYPES, index=s_idx)
                    with c_up2:
                        up_qty = st.number_input("Miktar", value=curr[5], min_value=1)
                        try: d_val = datetime.strptime(curr[8], "%Y-%m-%d").date()
                        except: d_val = datetime.now().date()
                        

                        guncel_ilk_gun = st.date_input("İlk Teslimat Tarihi", value=d_val)
                        
                        try: t_val = datetime.strptime(str(curr[7])[:5], "%H:%M").time()
                        except: t_val = datetime.now().time()
                        

                        guncel_saat = st.time_input("Saat", value=t_val)
                    
                    if st.form_submit_button("Güncelle"):
                        new_day = DAYS_TR_MAP[guncel_ilk_gun.weekday()]
                        baglanti = get_db_connection()
                        baglanti.execute("UPDATE members SET name=?, phone=?, sub_type=?, quantity=?, start_date=?, delivery_day=?, delivery_time=? WHERE id=?",
                                     (guncel_isim, guncel_tel, up_sub, up_qty, guncel_ilk_gun, new_day, str(guncel_saat), m_id))
                        baglanti.commit(); baglanti.close(); st.success("Güncellendi"); time.sleep(1); st.rerun()

    with st.expander("🗑️ Üye Sil", expanded=False):
        if not df_m.empty:
            sel_dels = st.multiselect("Silinecek Üyeler", list(member_dict.keys()))
            if sel_dels:
                
#ŞİFRE KORUMASI EKLENDİ yapayzeka desteği alındı 
                del_pass_m = st.text_input("Yönetici Şifresi", type="password", key="mem_del_pass")
                if st.button("Seçilenleri Sil"):
                    if check_password_match(del_pass_m, admin_pass_hash):
                        ids = [(member_dict[x],) for x in sel_dels]
                        baglanti = get_db_connection()
                        baglanti.executemany("DELETE FROM members WHERE id=?", ids)
                        baglanti.commit(); baglanti.close(); st.success("Silindi"); time.sleep(1); st.rerun()
                    else: st.error("Hatalı Şifre")


#AYARLAR
elif aktif_menu == "Ayarlar":
    st.title("⚙️ Ayarlar")
    baglanti = get_db_connection()
    st.subheader("Vergi Ayarı")
    try: current_tax = baglanti.execute("SELECT value FROM settings WHERE key='tax_rate'").fetchone()[0]
    except: current_tax = "8.0"
    new_tax = st.text_input("Vergi Oranı (%)", value=str(current_tax))
    if st.button("Vergiyi Güncelle"):
        baglanti.execute("INSERT OR REPLACE INTO settings (key, value) VALUES ('tax_rate', ?)", (new_tax,))
        baglanti.commit(); st.success("Güncellendi!"); time.sleep(1); st.rerun()

    st.divider(); st.subheader("🔐 Yönetici Şifresi")
    with st.expander("Şifre Değiştir"):
        cp = st.text_input("Mevcut", type="password")
        np = st.text_input("Yeni", type="password")
        if st.button("Değiştir"):
            sh = baglanti.execute("SELECT value FROM settings WHERE key='admin_pass'").fetchone()
            if sh and check_password_match(cp, sh[0]):
                baglanti.execute("UPDATE settings SET value=? WHERE key='admin_pass'", (make_hash(np),))
                baglanti.commit(); st.success("Tamam")
            else: st.error("Hata")

    st.divider(); st.subheader("⚠️ Sıfırlama")
    rp = st.text_input("Şifre", type="password", key="rp")
    if st.button("💣 SİL"):
        sh = baglanti.execute("SELECT value FROM settings WHERE key='admin_pass'").fetchone()
        if sh and check_password_match(rp, sh[0]):
            baglanti.execute("DELETE FROM products"); baglanti.execute("DELETE FROM members"); baglanti.execute("DELETE FROM delivery_logs"); baglanti.commit()
            st.success("Sıfırlandı")
        else: st.error("Şifre Yanlış")
    baglanti.close()
