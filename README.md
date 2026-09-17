import calendar
from datetime import date
import streamlit as st

st.set_page_config(page_title="Tính lãi tiền gửi tiết kiệm", page_icon="💰", layout="wide")


def add_months(d: date, months: int) -> date:
    """Cộng số tháng vào ngày d, tự điều chỉnh ngày cuối tháng."""
    month_index = d.month - 1 + months
    year = d.year + month_index // 12
    month = month_index % 12 + 1
    day = min(d.day, calendar.monthrange(year, month)[1])
    return date(year, month, day)


def add_term(d: date, term_value: int, term_unit: str) -> date:
    if term_unit == "Ngày":
        from datetime import timedelta
        return d + timedelta(days=term_value)
    if term_unit == "Tháng":
        return add_months(d, term_value)
    return add_months(d, term_value * 12)


def calc_interest(principal: float, annual_rate_pct: float, days: int) -> float:
    """Lãi theo Actual/365: gốc x lãi suất năm x số ngày / 365."""
    if days <= 0:
        return 0.0
    return principal * (annual_rate_pct / 100.0) * days / 365.0


def fmt_money(value: float) -> str:
    return f"{value:,.0f} đ".replace(",", ".")


def monthly_paid_before_withdrawal(
    principal: float,
    annual_rate_pct: float,
    term_start: date,
    term_end: date,
    withdrawal_date: date,
) -> float:
    """Tổng lãi kỳ hạn đã được chi theo tháng trước hoặc đúng ngày rút."""
    paid = 0.0
    cursor = term_start
    n = 1

    while True:
        payment_date = min(add_months(term_start, n), term_end)
        if payment_date > withdrawal_date or payment_date <= cursor:
            break
        days = (payment_date - cursor).days
        paid += calc_interest(principal, annual_rate_pct, days)
        cursor = payment_date
        if payment_date >= term_end:
            break
        n += 1

    return paid


def calculate_savings(
    principal: float,
    term_rate: float,
    non_term_rate: float,
    deposit_date: date,
    withdrawal_date: date,
    term_value: int,
    term_unit: str,
    payout_method: str,
):
    if withdrawal_date < deposit_date:
        raise ValueError("Ngày rút tiền không được trước ngày gửi tiền.")

    original_principal = principal
    current_principal = principal
    current_start = deposit_date

    total_eligible_interest = 0.0
    total_paid_before_withdrawal = 0.0
    completed_terms = 0
    term_days_earned = 0
    non_term_days = 0
    breakdown = []

    # Xử lý các kỳ đã hoàn tất. Ngày rút đúng ngày đáo hạn được xem là hoàn tất kỳ.
    while True:
        current_end = add_term(current_start, term_value, term_unit)
        if withdrawal_date < current_end:
            break

        days = (current_end - current_start).days
        interest = calc_interest(current_principal, term_rate, days)
        total_eligible_interest += interest
        term_days_earned += days
        completed_terms += 1

        if payout_method == "Nhận lãi trước":
            # Lãi của kỳ đã được trả ngay đầu kỳ.
            total_paid_before_withdrawal += interest
            end_balance = current_principal
        elif payout_method == "Nhận lãi hàng tháng":
            # Toàn bộ lãi kỳ hạn được chi dần trong kỳ.
            total_paid_before_withdrawal += interest
            end_balance = current_principal
        else:
            # Không rút tại ngày đáo hạn => nhập lãi vào gốc và tái tục.
            end_balance = current_principal + interest

        breakdown.append(
            {
                "Giai đoạn": f"Kỳ {completed_terms}",
                "Từ ngày": current_start.strftime("%d/%m/%Y"),
                "Đến ngày": current_end.strftime("%d/%m/%Y"),
                "Số ngày tính lãi": days,
                "Loại lãi suất": "Có kỳ hạn",
                "Gốc tính lãi": current_principal,
                "Tiền lãi": interest,
            }
        )

        current_principal = end_balance
        current_start = current_end

        # Rút đúng ngày đáo hạn: dừng, không mở thêm một kỳ mới.
        if withdrawal_date == current_start:
            break

        # Chống vòng lặp bất thường nếu người dùng nhập dữ liệu quá lớn.
        if completed_terms > 1000:
            raise ValueError("Số kỳ tái tục quá lớn. Vui lòng kiểm tra lại dữ liệu.")

    # Nếu rút sau ngày bắt đầu kỳ hiện tại nhưng trước đáo hạn => kỳ hiện tại bị rút trước hạn.
    partial_days = (withdrawal_date - current_start).days
    if partial_days > 0:
        non_term_days = partial_days
        partial_interest = calc_interest(current_principal, non_term_rate, partial_days)
        total_eligible_interest += partial_interest

        current_term_end = add_term(current_start, term_value, term_unit)

        if payout_method == "Nhận lãi trước":
            # Lãi kỳ hạn đầy đủ của kỳ hiện tại đã trả ở đầu kỳ, nhưng khi rút sớm
            # khách chỉ được hưởng lãi không kỳ hạn => phần chênh lệch bị bù trừ.
            full_term_days = (current_term_end - current_start).days
            upfront_paid = calc_interest(current_principal, term_rate, full_term_days)
            total_paid_before_withdrawal += upfront_paid
        elif payout_method == "Nhận lãi hàng tháng":
            total_paid_before_withdrawal += monthly_paid_before_withdrawal(
                current_principal,
                term_rate,
                current_start,
                current_term_end,
                withdrawal_date,
            )

        breakdown.append(
            {
                "Giai đoạn": "Kỳ đang rút trước hạn",
                "Từ ngày": current_start.strftime("%d/%m/%Y"),
                "Đến ngày": withdrawal_date.strftime("%d/%m/%Y"),
                "Số ngày tính lãi": partial_days,
                "Loại lãi suất": "Không kỳ hạn",
                "Gốc tính lãi": current_principal,
                "Tiền lãi": partial_interest,
            }
        )

    # Trường hợp nhận lãi trước và rút ngay trong ngày gửi: lãi đầu kỳ đã được trả,
    # nhưng số ngày thực hưởng là 0 nên phải hoàn lại toàn bộ phần lãi trước.
    elif withdrawal_date == deposit_date and payout_method == "Nhận lãi trước":
        first_end = add_term(deposit_date, term_value, term_unit)
        full_term_days = (first_end - deposit_date).days
        total_paid_before_withdrawal = calc_interest(principal, term_rate, full_term_days)

    # Tổng quyền lợi kinh tế = gốc ban đầu + tổng lãi hợp lệ.
    # Với lãi cuối kỳ có tái tục, phần lãi các kỳ trước đã nằm trong current_principal,
    # nhưng công thức này vẫn cho đúng tổng tiền khách nhận trong toàn bộ quá trình.
    total_customer_receives = original_principal + total_eligible_interest

    # Khoản nhận tại thời điểm rút = tổng quyền lợi - các khoản đã chi trước đó.
    amount_at_withdrawal = total_customer_receives - total_paid_before_withdrawal

    next_maturity = add_term(current_start, term_value, term_unit)

    if withdrawal_date == deposit_date:
        status = "Rút ngay ngày gửi"
    elif withdrawal_date == current_start and completed_terms > 0:
        status = "Rút đúng ngày đáo hạn"
    elif partial_days > 0:
        if completed_terms == 0:
            status = "Rút trước hạn"
        else:
            status = f"Rút trong kỳ tái tục lần {completed_terms}"
    else:
        status = "Đúng hạn"

    return {
        "original_principal": original_principal,
        "current_principal": current_principal,
        "total_interest": total_eligible_interest,
        "total_receives": total_customer_receives,
        "paid_before": total_paid_before_withdrawal,
        "amount_at_withdrawal": amount_at_withdrawal,
        "completed_terms": completed_terms,
        "term_days": term_days_earned,
        "non_term_days": non_term_days,
        "status": status,
        "current_term_start": current_start,
        "next_maturity": next_maturity,
        "breakdown": breakdown,
    }


st.title("💰 Ứng dụng tính lãi tiền gửi tiết kiệm")
st.caption(
    "Quy ước: lãi suất tính theo %/năm, cơ sở 365 ngày. "
    "Số ngày tính lãi từ ngày gửi đến trước ngày đáo hạn hoặc trước ngày rút tiền."
)

with st.form("savings_form"):
    col1, col2 = st.columns(2)

    with col1:
        principal = st.number_input(
            "Số tiền khách hàng gửi (VNĐ)",
            min_value=0.0,
            value=200_000_000.0,
            step=1_000_000.0,
            format="%.0f",
        )
        term_rate = st.number_input(
            "Lãi suất có kỳ hạn (%/năm)",
            min_value=0.0,
            value=5.0,
            step=0.1,
            format="%.3f",
        )
        non_term_rate = st.number_input(
            "Lãi suất không kỳ hạn (%/năm)",
            min_value=0.0,
            value=0.2,
            step=0.05,
            format="%.3f",
        )

    with col2:
        deposit_date = st.date_input("Ngày gửi tiền", value=date.today(), format="DD/MM/YYYY")
        withdrawal_date = st.date_input("Ngày rút tiền", value=date.today(), format="DD/MM/YYYY")

        c1, c2 = st.columns([1, 1])
        with c1:
            term_value = st.number_input("Kỳ hạn", min_value=1, value=3, step=1)
        with c2:
            term_unit = st.selectbox("Đơn vị kỳ hạn", ["Tháng", "Ngày", "Năm"])

    payout_method = st.radio(
        "Cách nhận tiền lãi",
        ["Nhận lãi trước", "Nhận lãi hàng tháng", "Nhận lãi cuối kỳ"],
        horizontal=True,
    )

    submitted = st.form_submit_button("🧮 TÍNH TOÁN", use_container_width=True, type="primary")

if submitted:
    if principal <= 0:
        st.error("Số tiền gửi phải lớn hơn 0.")
    elif withdrawal_date < deposit_date:
        st.error("Ngày rút tiền không được trước ngày gửi tiền.")
    else:
        try:
            result = calculate_savings(
                principal=principal,
                term_rate=term_rate,
                non_term_rate=non_term_rate,
                deposit_date=deposit_date,
                withdrawal_date=withdrawal_date,
                term_value=int(term_value),
                term_unit=term_unit,
                payout_method=payout_method,
            )

            st.success(f"Trạng thái khoản gửi: {result['status']}")

            a, b, c = st.columns(3)
            a.metric("Tiền gốc ban đầu", fmt_money(result["original_principal"]))
            b.metric("Tổng tiền lãi được hưởng", fmt_money(result["total_interest"]))
            c.metric("Tổng gốc + lãi", fmt_money(result["total_receives"]))

            d, e, f = st.columns(3)
            d.metric("Lãi đã nhận trước/định kỳ", fmt_money(result["paid_before"]))
            e.metric("Số tiền nhận tại ngày rút", fmt_money(result["amount_at_withdrawal"]))
            f.metric("Số kỳ đã hoàn tất", result["completed_terms"])

            st.info(
                f"**Số ngày hưởng lãi có kỳ hạn:** {result['term_days']} ngày  |  "
                f"**Số ngày hưởng lãi không kỳ hạn:** {result['non_term_days']} ngày"
            )

            if withdrawal_date > result["current_term_start"] and withdrawal_date < result["next_maturity"]:
                st.warning(
                    "Khách hàng rút trước ngày đáo hạn của kỳ hiện tại nên phần thời gian của kỳ này "
                    "được tính theo lãi suất không kỳ hạn. Nếu trước đó đã nhận lãi trước hoặc lãi hàng tháng "
                    "cao hơn mức được hưởng, phần chênh lệch được bù trừ vào số tiền nhận khi rút."
                )

            st.subheader("Chi tiết các giai đoạn tính lãi")
            if result["breakdown"]:
                display_rows = []
                for row in result["breakdown"]:
                    display_rows.append(
                        {
                            **row,
                            "Gốc tính lãi": fmt_money(row["Gốc tính lãi"]),
                            "Tiền lãi": fmt_money(row["Tiền lãi"]),
                        }
                    )
                st.dataframe(display_rows, use_container_width=True, hide_index=True)
            else:
                st.write("Chưa phát sinh ngày tính lãi.")

            with st.expander("Giải thích cách tính"):
                st.markdown(
                    f"""
- **Lãi có kỳ hạn:** `Gốc × {term_rate}% × Số ngày / 365`.
- **Lãi không kỳ hạn:** `Gốc × {non_term_rate}% × Số ngày / 365`.
- **Ngày tính lãi:** lấy chênh lệch giữa ngày bắt đầu và ngày đáo hạn/ngày rút, nên tự động không tính ngày cuối.
- **Tự động tái tục:** nếu qua ngày đáo hạn mà chưa rút, hệ thống mở kỳ mới có độ dài bằng kỳ hạn ban đầu.
- **Nhận lãi cuối kỳ:** lãi của kỳ hoàn tất được nhập vào gốc khi tự động tái tục.
- **Nhận lãi trước / hàng tháng:** lãi được coi là đã chi theo lịch; nếu rút sớm, hệ thống tính lại quyền lợi theo lãi không kỳ hạn và bù trừ phần chênh lệch.
"""
                )

        except Exception as exc:
            st.error(f"Không thể tính toán: {exc}")
