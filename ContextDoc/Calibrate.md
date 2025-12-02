# **Calibration Protocol: Implied Forward Logic for Volatility Surface**

## **1\. Problem Statement**

Issue: Large discrepancies are observed between Call and Put implied volatilities at the At-The-Money (ATM) strike, particularly for long-dated maturities.  
Root Cause: The theoretical forward price calculated using the fixed dividend (div) and interest rate (rf) curves ($F\_{theo} \= S\_0 e^{(r-q)T}$) diverges from the market-implied forward price ($F\_{mkt}$) embedded in option prices.  
Resolution: Switch to Implied Forward Calibration. We will solve for the forward price that satisfies Put-Call Parity at the ATM strike, thereby forcing the Call IV and Put IV to converge.

## **2\. Mathematical Logic**

### **Put-Call Parity**

$$C \- P \= e^{-rT}(F \- K)$$  
Instead of calculating $F$ from $r$ and $q$, we observe $C$ and $P$ at a strike $K\_{ref}$ (closest to ATM) and solve for $F\_{implied}$:  
$$F\_{implied} \= K\_{ref} \+ (C\_{ref} \- P\_{ref})e^{rT}$$

### **Implied Dividend Yield**

To remain compatible with the Black-Scholes solver (which expects $r$ and $q$), we convert $F\_{implied}$ into an **Implied Dividend Yield (**$q\_{imp}$**)**:  
$$q\_{imp} \= r \- \\frac{1}{T} \\ln\\left(\\frac{F\_{implied}}{S\_0}\\right)$$  
By using $q\_{imp}$ instead of the fixed curve $q(T)$, we force the model's forward to match the market.

## **3\. Implementation Steps**

### **Step 1: Modify Data Loading Loop**

*Location: The cell constructing points\_grid (approx Cell 164\)*  
Current Logic:  
Calculates r and q from curves, calculates F, then loops through options to get IV.  
**New Logic:**

1. Load Call and Put files.  
2. Find the "Reference Strike" ($K\_{ref}$): The strike where $|C\_{mid} \- P\_{mid}|$ is minimized, or simply the strike closest to Spot $S\_0$.  
3. Extract $C\_{ref}$ and $P\_{ref}$ (mid-prices) at this strike.  
4. Calculate F\_implied using the parity formula.  
5. Calculate q\_implied.  
6. Use q\_implied for all subsequent IV calculations for that maturity slice.

### **Step 2: Code Implementation Guide**

**Replace the variable initialization block inside the loop with this logic:**  
\# Inside the loop over files\[i\].items():  
r \= rf\[i\](T)

\# \--- NEW: IMPLIED FORWARD CALIBRATION \---  
\# 1\. Merge Calls and Puts on Strike to find the ATM pair  
df\_calls \= pd.read\_csv(path\_call)  
df\_puts \= pd.read\_csv(path\_put)

\# Filter for valid data first to avoid bad reference prices  
\# (Add your existing bid/ask validity checks here)

\# Find common strikes  
common\_strikes \= set(df\_calls\['Strike'\]).intersection(set(df\_puts\['Strike'\]))  
closest\_strike \= min(common\_strikes, key=lambda x: abs(x \- S0\[i\]))

\# Get Prices at Closest Strike  
c\_row \= df\_calls\[df\_calls\['Strike'\] \== closest\_strike\].iloc\[0\]  
p\_row \= df\_puts\[df\_puts\['Strike'\] \== closest\_strike\].iloc\[0\]

\# Calculate Mid Prices (or Last if Mid unavailable)  
\# ... \[Insert your existing price calculation logic here for these two rows\] ...  
C\_ref \= price\_call\_atm   
P\_ref \= price\_put\_atm

\# 2\. Calculate Implied Forward  
discount \= np.exp(-r \* T)  
F\_implied \= closest\_strike \+ (C\_ref \- P\_ref) / discount

\# 3\. Calculate Implied Dividend Yield (q\_imp)  
\# This q\_imp ensures: S0 \* exp((r \- q\_imp) \* T) \== F\_implied  
if T \> 0:  
    q\_implied \= r \- np.log(F\_implied / S0\[i\]) / T  
else:  
    q\_implied \= div\[i\](T) \# Fallback for T=0

\# Use this for the rest of the loop  
q \= q\_implied   
F \= F\_implied 

print(f"  T={T:.3f} | Spot={S0\[i\]:.2f} | F\_mkt={F:.2f} | q\_adj={q:.2%} (old q={div\[i\](T):.2%})")

\# \--- END NEW LOGIC \---

\# Proceed with existing OTM filtering and IV calculation...  
\# Ensure you strictly use F (the new F\_implied) for moneyness filtering:  
\# Calls: Keep if Strike \> F  
\# Puts:  Keep if Strike \< F

### **Step 3: Verify OTM Selection**

Ensure the selection logic relies on the **new** F.

* **OTM Puts:** if K \< F: ... (Using $F\_{implied}$)  
* **OTM Calls:** if K \> F: ... (Using $F\_{implied}$)

This ensures that the switch from Puts to Calls happens exactly at the market-neutral point, eliminating the vertical "step" in the volatility smile.

## **4\. Expected Outcome**

1. **ATM Convergence:** The IV calculated from the Call and the Put at the ATM strike will now be theoretically identical (or extremely close).  
2. **Smooth Transition:** The curve will flow smoothly from OTM Puts to OTM Calls.  
3. **Forward Skew:** You may notice the "center" of the smile shifts slightly compared to the fixed-curve version. This is the correct market behavior.