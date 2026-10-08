"""
Transmission Line Efficiency Calculator
---------------------------------------
A menu-driven program for engineering students to calculate the power loss
and efficiency of a three-phase transmission line, and to study how
efficiency changes with power factor, load and transmission voltage.

Transmission efficiency:
    Efficiency = Receiving-end power / Sending-end power * 100
               = Pr / (Pr + Ploss) * 100

Short line (I^2 R loss):
    Line current      I = Pr / (sqrt(3) * VL * pf)
    Line loss         Ploss = 3 * I^2 * R
    Efficiency        = Pr / (Pr + Ploss) * 100

Medium and long lines (ABCD constants, per phase):
    Vs = A * Vr + B * Ir          Is = C * Vr + D * Ir
    Sending-end power Ps = 3 * Re(Vs * conj(Is))
    Efficiency        = Pr / Ps * 100
    Nominal-pi:  A = D = 1 + ZY/2,  B = Z,  C = Y(1 + ZY/4)
    Long line:   A = D = cosh(gl),  B = Zc*sinh(gl),  C = sinh(gl)/Zc

Key ideas:
    Higher power factor  -> lower current -> lower loss -> higher efficiency
    Higher voltage       -> lower current -> lower loss -> higher efficiency
"""

import cmath
import math

SQRT3 = math.sqrt(3)


# ---------------------------------------------------------------- input helpers
def get_positive_float(prompt):
    """Keep asking until the user enters a valid positive number."""
    while True:
        try:
            value = float(input(prompt))
            if value <= 0:
                print("  Please enter a value greater than zero.")
                continue
            return value
        except ValueError:
            print("  Invalid input. Please enter a number.")


def get_nonneg_float(prompt):
    """Ask for a number that may be zero but not negative."""
    while True:
        try:
            value = float(input(prompt))
            if value < 0:
                print("  Please enter zero or a positive value.")
                continue
            return value
        except ValueError:
            print("  Invalid input. Please enter a number.")


def get_power_factor():
    """Ask for a power factor between 0 (exclusive) and 1 (inclusive)."""
    while True:
        pf = get_positive_float("Load power factor (0 to 1, lagging): ")
        if pf <= 1:
            return pf
        print("  Power factor cannot be greater than 1.")


def get_line_data(need_shunt=False):
    """Ask for the line parameters (per phase, per km)."""
    length = get_positive_float("Line length (km): ")
    r = get_positive_float("Resistance r (ohm/km): ")
    x = get_positive_float("Inductive reactance x (ohm/km): ")
    if need_shunt:
        b = get_positive_float("Shunt susceptance b (microsiemens/km): ")
    else:
        b = get_nonneg_float("Shunt susceptance b (microsiemens/km, 0 if ignored): ")
    return length, r, x, b


# ------------------------------------------------------------------ core maths
def choose_model(length):
    """Pick the line model from the line length."""
    if length < 80:
        return "short"
    if length <= 250:
        return "pi"
    return "long"


def abcd_constants(model, length, r, x, b_us):
    """Return the complex A, B, C, D constants for the chosen line model."""
    z_per_km = complex(r, x)
    y_per_km = complex(0, b_us * 1e-6)
    z_total = z_per_km * length
    y_total = y_per_km * length

    if model == "short" or b_us == 0:
        return 1 + 0j, z_total, 0j, 1 + 0j

    if model == "pi":
        a = 1 + z_total * y_total / 2
        return a, z_total, y_total * (1 + z_total * y_total / 4), a

    gamma = cmath.sqrt(z_per_km * y_per_km)
    zc = cmath.sqrt(z_per_km / y_per_km)
    gl = gamma * length
    a = cmath.cosh(gl)
    return a, zc * cmath.sinh(gl), cmath.sinh(gl) / zc, a


def line_current(p_mw, v_kv, pf):
    """I = P / (sqrt(3) * VL * pf)  in amperes."""
    return p_mw * 1e6 / (SQRT3 * v_kv * 1000 * pf)


def efficiency_from_loss(p_recv, p_loss):
    """Efficiency (%) = Pr / (Pr + Ploss) * 100"""
    return p_recv / (p_recv + p_loss) * 100


def solve_line(abcd, v_kv, p_mw, pf):
    """Solve a line with a lagging load. Returns a dictionary of results."""
    a, b, c, d = abcd
    vr = complex(v_kv * 1000 / SQRT3, 0)
    ir_mag = line_current(p_mw, v_kv, pf)
    ir = cmath.rect(ir_mag, -math.acos(pf))

    vs = a * vr + b * ir
    i_s = c * vr + d * ir

    p_send = 3 * (vs * i_s.conjugate()).real / 1e6
    vr_no_load = abs(vs) / abs(a)
    return {
        "current": ir_mag,
        "p_send": p_send,
        "loss": p_send - p_mw,
        "efficiency": p_mw / p_send * 100,
        "regulation": (vr_no_load - abs(vr)) / abs(vr) * 100,
        "vs_kv": SQRT3 * abs(vs) / 1000,
    }


# ------------------------------------------------------------------- display
def short_line_efficiency():
    length = get_positive_float("Line length (km): ")
    r = get_positive_float("Resistance r (ohm/km): ")
    v_kv = get_positive_float("Receiving-end line voltage (kV): ")
    p_mw = get_positive_float("Receiving-end load power (MW, three-phase): ")
    pf = get_power_factor()

    r_total = r * length
    i = line_current(p_mw, v_kv, pf)
    loss = 3 * i ** 2 * r_total / 1e6
    eff = efficiency_from_loss(p_mw, loss)

    print("\n  ----- Short Line Efficiency -----")
    print(f"  Total line resistance   = {r_total:.3f} ohm per phase")
    print(f"  Line current            = {i:.2f} A")
    print(f"  Line power loss (3I^2R) = {loss:.4f} MW")
    print(f"  Sending-end power       = {p_mw + loss:.4f} MW")
    print(f"  Loss as % of load       = {loss / p_mw * 100:.2f} %")
    print(f"  TRANSMISSION EFFICIENCY = {eff:.2f} %")


def abcd_efficiency(model, title):
    length, r, x, b = get_line_data(need_shunt=True)
    v_kv = get_positive_float("Receiving-end line voltage (kV): ")
    p_mw = get_positive_float("Receiving-end load power (MW, three-phase): ")
    pf = get_power_factor()

    res = solve_line(abcd_constants(model, length, r, x, b), v_kv, p_mw, pf)

    print(f"\n  ----- {title} -----")
    print(f"  Receiving-end current   = {res['current']:.2f} A")
    print(f"  Sending-end voltage     = {res['vs_kv']:.3f} kV")
    print(f"  Sending-end power       = {res['p_send']:.4f} MW")
    print(f"  Line power loss         = {res['loss']:.4f} MW")
    print(f"  Voltage regulation      = {res['regulation']:.2f} %")
    print(f"  TRANSMISSION EFFICIENCY = {res['efficiency']:.2f} %")


def table_header(first_col):
    print(f"\n  {first_col:>14}{'I (A)':>10}{'Loss (MW)':>12}{'Eff (%)':>10}{'VR (%)':>10}")


def table_row(label, res):
    print(f"  {label:>14}{res['current']:>10.1f}{res['loss']:>12.4f}{res['efficiency']:>10.2f}{res['regulation']:>10.2f}")


def efficiency_vs_power_factor():
    length, r, x, b = get_line_data(need_shunt=True)
    v_kv = get_positive_float("Receiving-end line voltage (kV): ")
    p_mw = get_positive_float("Receiving-end load power (MW, three-phase): ")

    model = choose_model(length)
    abcd = abcd_constants(model, length, r, x, b)

    print("\n  ----- Efficiency vs Power Factor -----")
    table_header("Power factor")
    for pf in (0.6, 0.7, 0.8, 0.9, 0.95, 1.0):
        table_row(f"{pf:.2f}", solve_line(abcd, v_kv, p_mw, pf))
    print("  Low power factor needs more current for the same power, so losses rise.")


def efficiency_vs_load():
    length, r, x, b = get_line_data(need_shunt=True)
    v_kv = get_positive_float("Receiving-end line voltage (kV): ")
    p_max = get_positive_float("Maximum load power (MW): ")
    pf = get_power_factor()

    model = choose_model(length)
    abcd = abcd_constants(model, length, r, x, b)

    print("\n  ----- Efficiency vs Load -----")
    table_header("Load (MW)")
    for fraction in (0.2, 0.4, 0.6, 0.8, 1.0):
        table_row(f"{p_max * fraction:.2f}", solve_line(abcd, v_kv, p_max * fraction, pf))
    print("  Loss grows with the square of the current, so efficiency falls at heavy load.")


def efficiency_vs_voltage():
    length, r, x, b = get_line_data(need_shunt=True)
    p_mw = get_positive_float("Load power to be transmitted (MW, three-phase): ")
    pf = get_power_factor()

    model = choose_model(length)
    abcd = abcd_constants(model, length, r, x, b)

    print("\n  ----- Efficiency vs Transmission Voltage -----")
    table_header("Voltage (kV)")
    for v_kv in (33, 66, 110, 132, 220, 400):
        table_row(f"{v_kv}", solve_line(abcd, v_kv, p_mw, pf))
    print("  Same power at a higher voltage means lower current and lower loss.")
    print("  (Same conductor assumed. Very high VR means that voltage is not practical.)")


def menu():
    print("\n" + "=" * 58)
    print("     TRANSMISSION LINE EFFICIENCY CALCULATOR")
    print("=" * 58)
    print(" 1. Short line efficiency (I^2 R loss)")
    print(" 2. Medium line efficiency (nominal-pi)")
    print(" 3. Long line efficiency")
    print(" 4. Efficiency vs load power factor")
    print(" 5. Efficiency vs load (MW)")
    print(" 6. Efficiency vs transmission voltage")
    print(" 0. Exit")
    print("-" * 58)


def main():
    while True:
        menu()
        choice = input("Enter your choice: ").strip()

        if choice == "1":
            short_line_efficiency()
        elif choice == "2":
            abcd_efficiency("pi", "Medium Line Efficiency (nominal-pi)")
        elif choice == "3":
            abcd_efficiency("long", "Long Line Efficiency")
        elif choice == "4":
            efficiency_vs_power_factor()
        elif choice == "5":
            efficiency_vs_load()
        elif choice == "6":
            efficiency_vs_voltage()
        elif choice == "0":
            print("\nThank you for using the calculator. Goodbye!")
            break
        else:
            print("  Invalid choice. Please select from the menu.")


if __name__ == "__main__":
    main()
