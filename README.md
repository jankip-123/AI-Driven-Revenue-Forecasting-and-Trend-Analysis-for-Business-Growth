import streamlit as st
import pandas as pd
import os
from otp_utils import send_email_otp
from streamlit_option_menu import option_menu
import random
import string
import csv
import plotly.express as px
from datetime import timedelta
import matplotlib.pyplot as plt
from sklearn.metrics import mean_absolute_error
from sklearn.metrics import mean_absolute_error, mean_squared_error, r2_score
import streamlit as st
import pandas as pd
import plotly.express as px
import streamlit_authenticator as stauth




# ===================== CONFIG =====================
USER_CSV = "users.csv"
st.set_page_config(page_title="AI Revenue Forecasting", layout="wide")

# ===================== HELPER FUNCTIONS =====================
def generate_code(length=6):
    return ''.join(random.choices(string.ascii_uppercase + string.digits, k=length))

def send_otp(email):
    otp = generate_code()
    st.session_state[f"otp_{email}"] = otp
    success = send_email_otp(email, otp)
    if success:
        st.success(f"OTP sent to {email}")
    else:
        st.error("Failed to send OTP. Please check your email configuration.")

def save_user_to_csv(username, email, password):
    file_exists = os.path.isfile(USER_CSV)
    with open(USER_CSV, mode='a', newline='') as file:
        writer = csv.writer(file)
        if not file_exists:
            writer.writerow(["Username", "Email", "Password"])
        writer.writerow([username, email, password])

# ===================== SESSION INITIALIZATION =====================
if "logged_in" not in st.session_state:
    st.session_state.logged_in = False
if "username" not in st.session_state:
    st.session_state.username = ""
if "Username" not in st.session_state:
    st.session_state.Username = ""
if "selected_tab" not in st.session_state:
    st.session_state.selected_tab = "Home"
# Initialize session state if it does not exist
if 'authenticated' not in st.session_state:
    st.session_state.authenticated = False  # Default value, adjust as needed
rerun = st.experimental_rerun if hasattr(st, "experimental_rerun") else st.rerun

# ===================== Query Params =====================
query_params = st.query_params
if "username" in query_params and "auth" in query_params:
    if query_params["auth"] == "true":
        st.session_state.authenticated = True
        st.session_state.username = query_params["username"]
if "tab" in query_params:
    st.session_state.selected_tab = query_params["tab"]

# ===================== Sidebar =====================
with st.sidebar:
    if not st.session_state.authenticated:
        selected = option_menu(
            "RevTrend AI",
            ["Login", "Register"],
            icons=["lock", "key"],
            menu_icon="graph-up",
            default_index=["Login", "Register"].index(st.session_state.selected_tab) if st.session_state.selected_tab in ["Login", "Register"] else 0,
            key="menu_key"
        )
    else:
        menu_options = [
            "Home", "Data Ingestion & Preprocessing", "Trend Analysis & Visualization","Sales Analysis",
            "Revenue Forecasting", "Anomaly Detection",
            "User Dashboard", "Settings", "Help & Documentation", "About Us", "Log Out"
        ]
        selected = option_menu(
            "Dashboard",
            menu_options,
            icons=["house", "upload", "graph-up", "bar-chart-line", "calendar-check", "exclamation-triangle",
                   "person-circle", "gear",
                   "question-circle", "info-circle", "box-arrow-right"],
            menu_icon="graph-up",
            default_index=menu_options.index(st.session_state.selected_tab) if st.session_state.selected_tab in menu_options else 0,
            key="menu_key"
        )

if selected != st.session_state.selected_tab:
    st.session_state.selected_tab = selected
    st.query_params.update({"tab": selected})

# ===================== Login =====================
if not st.session_state.authenticated and selected == "Login":

    st.subheader("Login to Your Account")

    login_email = st.text_input("Email", key="login_email")
    st.text_input("Password", type="password", key="login_pass")
    login_role = st.selectbox("Select Role", ["Admin", "Analyst", "Manager"])
    st.checkbox("I'm not a robot")
    st.markdown("<a href='#' style='font-size: 14px;'>Forgot password?</a>", unsafe_allow_html=True)

    if st.button("Login"):
        if login_email and st.session_state.login_pass:
            st.session_state.logged_in = True
            st.session_state.username = login_email.split('@')[0]
            st.session_state.selected_tab = "Home"
            st.query_params.update({
                "username": st.session_state.username,
                "auth": "true",
                "tab": st.session_state.selected_tab
            })
            st.success(f"Welcome back, {login_role}!")
            rerun()
        else:
            st.error("Please enter email and password.")

# ===================== Register =====================
elif not st.session_state.authenticated and selected == "Register":
    st.subheader("Create an Account")

    # Registration input fields
    st.text_input("Username", key="reg_name")
    reg_email = st.text_input("Email", key="register_email")
    st.text_input("Password", type="password", key="reg_pass")
    st.text_input("Confirm Password", type="password", key="reg_confirm")

    if st.button("Send OTP"):
        if reg_email:
            send_otp(reg_email)
        else:
            st.warning("Please enter your email first.")

    otp_entered = st.text_input("Enter the OTP sent to your email", key="reg_otp")
    correct_otp = st.session_state.get(f"otp_{reg_email}")

    if "captcha_value" not in st.session_state:
        st.session_state.captcha_value = generate_code()

    st.markdown(f"**CAPTCHA**: `{st.session_state.captcha_value}`")
    if st.button("Refresh CAPTCHA"):
        st.session_state.captcha_value = generate_code()

    captcha_input = st.text_input("Enter CAPTCHA here", key="captcha_input")

    if st.button("Register"):
        if not all([
            st.session_state.reg_name,
            reg_email,
            st.session_state.reg_pass,
            st.session_state.reg_confirm,
            otp_entered,
            captcha_input
        ]):
            st.error("Please fill all fields.")
        elif st.session_state.reg_pass != st.session_state.reg_confirm:
            st.error("Passwords do not match.")
        elif otp_entered != correct_otp:
            st.error("Entered OTP does not match.")
        elif captcha_input != st.session_state.captcha_value:
            st.error("Entered CAPTCHA is incorrect.")
        else:
            # Save user data and set username to session state
            save_user_to_csv(
                username=st.session_state.reg_name,
                email=reg_email,
                password=st.session_state.reg_pass
            )
            st.session_state.Username = st.session_state.reg_name  # Store the username after successful registration
            st.success("✅ Registration successful! You can now log in.")

# ===================== Authenticated Pages =====================
elif st.session_state.authenticated:
    if selected == "Home":

        if "Username" in st.session_state:
            st.markdown(
                f'<div style="text-align: center; margin-bottom: 0;"><h3 style="color:#A020F0; margin-bottom: 0;">Welcome, <b>{st.session_state.Username}</b>!</h3></div>',
                unsafe_allow_html=True
            )
        else:
            st.warning("Username not found. Please log in again.")

        st.markdown(
            f"""
            <style>
            .center {{
                display: flex;
                flex-direction: column;
                align-items: center;
                justify-content: flex-start; /* Align items to the top */
            }}
            .hero {{
                font-size: 36px;
                font-weight: 700;
                margin-top: 10px; /* Reduced top margin */
                color: #1f77b4;
                text-align: center;
            }}
            .subtext {{
                 font-size: 18px;
                 font-weight: 400;
                 color: #CCCCCC; /* Light gray text for dark theme */
                 text-align: center;
                 max-width: 900px;
                 line-height: 1.6;
                 font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
                 margin-bottom: 40px; /* 👈 Space below subtext */
            }}
            </style>
            <div class="center">
                <div class="hero">AI-Driven Revenue Forecasting Platform</div>
                <div class="subtext">
                    Predict the future of your business with precision.<br>
                    Our AI-powered system analyzes trends, detects anomalies, and forecasts your revenue with smart insights.
                </div>
            </div>
            """,
            unsafe_allow_html=True
        )

        st.image("AI_Driven.png", use_container_width=True)

        st.markdown("### Why Choose Our Platform?")

        st.markdown("""
        <style>
        .icon-card {
            background-color: #ffffff;
            border-radius: 15px;
            padding: 30px 20px;
            box-shadow: 0 4px 16px rgba(0, 0, 0, 0.06);
            text-align: center;
            transition: all 0.3s ease;
            height: 220px;
        }
        .icon-card:hover {
            transform: translateY(-6px);
            box-shadow: 0 6px 18px rgba(0, 0, 0, 0.12);
        }
        .icon-img {
            font-size: 42px;
            margin-bottom: 12px;
        }
        .card-title {
            font-weight: 700;
            font-size: 20px;
            margin-bottom: 10px;
            color: #1a1a1a;
        }
        .card-text {
            font-size: 15px;
            color: #4a4a4a;
            line-height: 1.4;
        }
        </style>
        """, unsafe_allow_html=True)

        col1, col2, col3 = st.columns(3)

        with col1:
            st.markdown("""
            <div class="icon-card">
                <div class="icon-img">📈</div>
                <div class="card-title">Smart Forecasting</div>
                <div class="card-text">
                    Leverage ML models to forecast future<br> revenue trends with confidence.
                </div>
            </div>
            """, unsafe_allow_html=True)

        with col2:
            st.markdown("""
            <div class="icon-card">
                <div class="icon-img">📊</div>
                <div class="card-title">Real-Time Trends</div>
                <div class="card-text">
                    Visual dashboards to monitor growth<br> and discover new patterns.
                </div>
            </div>
            """, unsafe_allow_html=True)

        with col3:
            st.markdown("""
            <div class="icon-card">
                <div class="icon-img">🚨</div>
                <div class="card-title">Anomaly Alerts</div>
                <div class="card-text">
                    Detect unexpected patterns early<br> to avoid business disruptions.
                </div>
            </div>
            """, unsafe_allow_html=True)
        st.markdown("<br>", unsafe_allow_html=True)
        st.markdown("### How It Works")
        st.markdown("""
        - **Upload Your Historical Revenue Data**  
        - **System Cleans & Prepares the Data**  
        - **ML Models Predict Future Trends**  
        - **Interactive Dashboards Display Insights**  
        - **Download Reports & Take Action**  
        """)

        st.markdown("---")

        st.markdown("### Who Is This For?")
        st.markdown("""
        - Retailers aiming to manage seasonal demand  
        - E-commerce businesses forecasting monthly revenue  
        - SaaS platforms tracking recurring income  
        - Financial analysts and business decision-makers  
        """)

        st.markdown("---")

        st.markdown("### Secure & Role-Based Access")
        st.write("Log in as an **Admin**, **Analyst**, or **Manager** to access tools tailored to your role.")

        st.success(
            "Ready to explore? Use the sidebar to navigate through modules like Forecasting, Trend Analysis, and Reports.")

#---------------------------------------------------------------------------------------------------------------------------------
    elif selected == "Data Ingestion & Preprocessing":

        st.markdown(
            f"""
            <style>
            .center {{
                display: flex;
                flex-direction: column;
                align-items: center;
                justify-content: flex-start;
            }}
            .hero {{
                font-size: 32px;
                font-weight: 700;
                margin-top: 10px;
                color: #1f77b4;
                text-align: center;
            }}
            .subtext {{
                 font-size: 18px;
                 font-weight: 400;
                 color: #CCCCCC;
                 text-align: center;
                 max-width: 900px;
                 line-height: 1.6;
                 font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
                 margin-bottom: 40px;
            }}
            </style>
            <div class="center">
                <div class="hero">Data Ingestion & Preprocessing Module</div>
                <div class="subtext">
                    Upload, clean, and prepare your data effortlessly.<br>
                    This module ensures your revenue data is formatted correctly, detects missing values, and readies it for deep AI analysis.
                </div>
            </div>
            """,
            unsafe_allow_html=True
        )

        st.markdown("""
            <style>
                .block-container div[data-testid="stFileUploader"] {
                    margin-top: -20px
                }
            </style>
        """, unsafe_allow_html=True)
        st.markdown("""
            <h3 style="font-size:20px">Upload your <strong>historical revenue data</strong> (CSV format):</h3>
        """, unsafe_allow_html=True)

        uploaded_file = st.file_uploader("", type=["csv"])

        if uploaded_file is not None:
            import pandas as pd
            import numpy as np
            from scipy.stats import zscore

            # Read the CSV
            try:
                df = pd.read_csv(uploaded_file)

                st.success("File uploaded successfully!")
                st.dataframe(df.head(10), use_container_width=True)

                st.markdown("""
                    <h3 style='font-size:26px; font-weight: bold; color:#1E90FF;'>
                        Preprocessing Steps
                    </h3>
                """, unsafe_allow_html=True)

                st.markdown("""
                    <h4 style='font-size:22px; font-weight:bold; padding:8px 12px; background-color:#f0f2f6; 
                    border-left: 6px solid #1E90FF; border-radius: 6px; color: #333;'>
                        1️⃣ Handling Missing Values
                    </h4>
                """, unsafe_allow_html=True)
                st.markdown("<br>", unsafe_allow_html=True)


                missing = df.isnull().sum()
                missing = missing[missing > 0]
                if not missing.empty:
                    total_missing = missing.sum()
                    st.warning(f"Missing values detected: {total_missing} missing values in total")
                    st.dataframe(missing.to_frame(name="Missing Count"))

                    # Automatically drop missing values
                    df.dropna(inplace=True)
                    st.success(f"Missing values removed. Remaining rows: {len(df)}")
                else:
                    st.success("Great! Your dataset has no missing values. Ready for the next step!")


                # Directly apply date formatting
                st.markdown("""
                    <h4 style='font-size:22px; font-weight:bold; padding:8px 12px; background-color:#f0f2f6; 
                    border-left: 6px solid #1E90FF; border-radius: 6px; color: #333;'>
                        2️⃣ Date Formatting
                    </h4>
                """, unsafe_allow_html=True)
                st.markdown("<br>", unsafe_allow_html=True)
                # Show original 'Date' column (as it exists before conversion)
                if 'Date' in df.columns:
                    st.write("**Original 'Date' column preview:**")
                    st.dataframe(df[['Date']].head())

                    try:
                        # Convert to datetime
                        df['Date'] = pd.to_datetime(df['Date'])

                        st.success("Date formatting applied successfully!")

                        # Format and display date only in dd-mm-YYYY format
                        df['Formatted Date'] = df['Date'].dt.strftime('%d-%m-%Y')

                        st.write("**Formatted 'Date' column (dd-mm-YYYY):**")
                        st.dataframe(df[['Formatted Date']].head())

                    except Exception as e:
                        st.error("Failed to convert 'Date' column to datetime. Please check format.")
                        st.exception(e)
                else:
                    st.error("'Date' column not found in the uploaded dataset.")

                # Outlier Detection using Z-Score
                st.markdown("""
                    <h4 style='font-size:22px; font-weight:bold; padding:8px 12px; background-color:#f0f2f6; 
                    border-left: 6px solid #1E90FF; border-radius: 6px; color: #333;'>
                        3️⃣ Outlier Detection using Z-Score
                    </h4>
                """, unsafe_allow_html=True)
                st.markdown("<br>", unsafe_allow_html=True)
                numeric_cols = df.select_dtypes(include=[np.number]).columns.tolist()

                # Assuming 'Revenue' is the column to detect outliers
                z_scores = zscore(df['Revenue'])
                abs_z_scores = np.abs(z_scores)
                outliers = (abs_z_scores > 3)
                num_outliers = np.sum(outliers)

                st.write(f"Detected {num_outliers} outliers using Z-score.")

                if st.checkbox("Remove Outliers?"):
                    df = df[~outliers]
                    st.success(f"Outliers removed. Remaining rows: {len(df)}")

                st.markdown("### Cleaned Data Preview")
                st.dataframe(df.head(10), use_container_width=True)

                # Option to download cleaned data
                csv = df.to_csv(index=False).encode('utf-8')
                st.download_button("Download Cleaned Data as CSV", data=csv, file_name="cleaned_revenue_data.csv",
                                   mime="text/csv")

            except Exception as e:
                st.error(f"Something went wrong while processing the file: {e}")
        else:
            st.info("Please upload a CSV file to begin.")

#----------------------------------------------------------------------------------------------------------------------------------

    elif selected == "Sales Analysis":

        st.markdown(
            f"""
            <style>
            .center {{
                display: flex;
                flex-direction: column;
                align-items: center;
                justify-content: flex-start;
            }}
            .hero {{
                font-size: 32px;
                font-weight: 700;
                margin-top: 10px;
                color: #1f77b4;
                text-align: center;
            }}
            .subtext {{
                 font-size: 18px;
                 font-weight: 400;
                 color: #CCCCCC;
                 text-align: center;
                 max-width: 900px;
                 line-height: 1.6;
                 font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
                 margin-bottom: 40px;
            }}
            </style>
            <div class="center">
                <div class="hero">Sales Analysis</div>
                <div class="subtext">
                    Dive deep into your sales data to understand performance drivers.<br>
                    Analyze top products, regions, time periods, and more to unlock insights for growth.
                </div>
            </div>
            """,
            unsafe_allow_html=True
        )




        @st.cache_data
        def load_data():

            df = pd.read_csv("AI_revenue_forecasting_dataset.csv")

            df["Date"] = pd.to_datetime(df["Date"])

            df["Month"] = df["Date"].dt.to_period("M").astype(str)

            return df


        df = load_data()



        # --- 2. Top Products ---

        st.markdown("""
            <h4 style='font-size:22px; font-weight:bold; padding:8px 12px; background-color:#f0f2f6; 
            border-left: 6px solid #FF8C00; border-radius: 6px; color: #333;'>
                🏆 Top Selling Products
            </h4>
        """, unsafe_allow_html=True)

        top_products = df.groupby("Subcategory")["Revenue"].sum().sort_values(ascending=False).reset_index()

        fig = px.bar(top_products.head(10), x="Revenue", y="Subcategory", orientation='h',

                     title="Top 10 Subcategories by Revenue", color='Revenue', text_auto='.2s')

        fig.update_layout(yaxis=dict(autorange="reversed"))

        st.plotly_chart(fig, use_container_width=True)

        # --- 3. Best Performing Region ---

        st.markdown("""
            <h4 style='font-size:22px; font-weight:bold; padding:8px 12px; background-color:#f0f2f6; 
            border-left: 6px solid #32CD32; border-radius: 6px; color: #333;'>
                🌍 Region-wise Sales Distribution
            </h4>
        """, unsafe_allow_html=True)

        region_rev = df.groupby("Region")["Revenue"].sum().reset_index()

        fig = px.pie(region_rev, names="Region", values="Revenue", title="Revenue by Region")

        st.plotly_chart(fig, use_container_width=True)

        # --- 4. Top Customer Type ---

        st.markdown("""
            <h4 style='font-size:22px; font-weight:bold; padding:8px 12px; background-color:#f0f2f6; 
            border-left: 6px solid #8A2BE2; border-radius: 6px; color: #333;'>
                👥 Revenue by Customer Type
            </h4>
        """, unsafe_allow_html=True)

        cust_rev = df.groupby("Customer_Type")["Revenue"].sum().reset_index()

        fig = px.pie(cust_rev, names="Customer_Type", values="Revenue", title="Revenue by Customer Type")

        st.plotly_chart(fig, use_container_width=True)

        # --- 5. Daily Sales Trend ---

        # Ensure Date is datetime and Revenue is numeric
        df["Date"] = pd.to_datetime(df["Date"], errors='coerce')
        df["Revenue"] = pd.to_numeric(df["Revenue"], errors='coerce')

        # Drop rows with missing dates or revenue
        df = df.dropna(subset=["Date", "Revenue"])

        # Group by date
        daily_sales = df.groupby("Date", as_index=False)["Revenue"].sum()

        # Section header
        st.markdown("""
            <h4 style='font-size:22px; font-weight:bold; padding:10px 15px; background-color:#e6f0fa; 
            border-left: 6px solid #0077cc; border-radius: 5px; color: #222; margin-top: 30px;'>
                📅 Daily Sales Trend
            </h4>
        """, unsafe_allow_html=True)

        # Create Plotly line chart
        fig = px.line(
            daily_sales,
            x="Date",
            y="Revenue",
            markers=True,
            template="plotly_white",
            labels={"Date": "Date", "Revenue": "Revenue"},
        )

        fig.update_traces(line=dict(color='#0077cc', width=3), marker=dict(size=6, color="#ff7f0e"))

        fig.update_layout(
            xaxis_title="Date",
            yaxis_title="Revenue",
            showlegend=False,
            margin=dict(t=20, b=40, l=0, r=0),
            height=420
        )

        # Show the plot
        st.plotly_chart(fig, use_container_width=True)
        # --- 6. Monthly Sales Trend ---

        st.markdown("""
            <h4 style='font-size:22px; font-weight:bold; padding:8px 12px; background-color:#f0f2f6; 
            border-left: 6px solid #20B2AA; border-radius: 6px; color: #333;'>
                🗓️ Monthly Sales Trend
            </h4>
        """, unsafe_allow_html=True)

        monthly_sales = df.groupby("Month")["Revenue"].sum().reset_index()

        fig = px.area(monthly_sales, x="Month", y="Revenue", title="Monthly Revenue Trend", line_shape="spline")

        st.plotly_chart(fig, use_container_width=True)

        # --- 7. Sales Table View ---

        st.markdown("""
            <h4 style='font-size:22px; font-weight:bold; padding:8px 12px; background-color:#f0f2f6; 
            border-left: 6px solid #DAA520; border-radius: 6px; color: #333;'>
                📋 Sales Summary Table
            </h4>
        """, unsafe_allow_html=True)

        st.markdown("<br>", unsafe_allow_html=True)

        summary_table = df.groupby(["Category", "Subcategory", "Region", "Customer_Type"]).agg({

            "Revenue": "sum",

            "Quantity_Sold": "sum"

        }).reset_index().sort_values("Revenue", ascending=False)

        st.dataframe(summary_table, use_container_width=True)
#---------------------------------------------------------------------------------------------------------------------

    elif selected == "Trend Analysis & Visualization":
        # Load data
        @st.cache_data
        def load_data():
            df = pd.read_csv("AI_revenue_forecasting_dataset.csv")
            df["Date"] = pd.to_datetime(df["Date"])
            return df


        df = load_data()

        st.markdown(
            f"""
            <style>
            .center {{
                display: flex;
                flex-direction: column;
                align-items: center;
                justify-content: flex-start;
            }}
            .hero {{
                font-size: 32px;
                font-weight: 700;
                margin-top: 10px;
                color: #1f77b4;
                text-align: center;
            }}
            .subtext {{
                 font-size: 18px;
                 font-weight: 400;
                 color: #CCCCCC;
                 text-align: center;
                 max-width: 900px;
                 line-height: 1.6;
                 font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
                 margin-bottom: 40px;
            }}
            </style>
            <div class="center">
                <div class="hero">Trend Analysis & Visualization</div>
                <div class="subtext">
                    Discover patterns and gain insights from your historical revenue data.<br>
                    This module highlights key trends, seasonal effects, and anomalies with interactive charts and visual summaries.
                </div>
            </div>
            """,
            unsafe_allow_html=True
        )

        # Tabs for each type of visualization
        tabs = st.tabs([
            "Category Revenue", "Region Revenue", "Channel Performance",
            "Promotion Impact", "Customer Types", "Seasonality", "Anomaly Trends"
        ])

        # --- Category/Subcategory Revenue ---
        with tabs[0]:
            st.subheader("💼 Revenue by Product Category and Subcategory")
            st.write(
                "This bar chart helps you understand which product categories and subcategories are generating the highest revenue.")
            cat_rev = df.groupby(["Category", "Subcategory"])["Revenue"].sum().reset_index().sort_values("Revenue",
                                                                                                         ascending=False)
            fig = px.bar(cat_rev, x='Subcategory', y='Revenue', color='Category', text_auto=True,
                         title='Revenue by Product Category & Subcategory')
            fig.update_layout(xaxis_tickangle=-45)
            st.plotly_chart(fig, use_container_width=True)
            # Assuming cat_rev has already been grouped and calculated
            cat_rev_sorted = cat_rev.sort_values(["Category", "Revenue"], ascending=[True, False])

            # Create a new dataframe to hold the final result
            formatted_data = []

            # Loop through each category and subcategory
            for category in cat_rev_sorted["Category"].unique():
                # Filter subcategories for this category
                subcategories = cat_rev_sorted[cat_rev_sorted["Category"] == category]
                for index, row in subcategories.iterrows():
                    formatted_data.append([row['Category'], row['Subcategory'], row["Revenue"]])

            # Create the final dataframe
            formatted_df = pd.DataFrame(formatted_data, columns=["Category", "Subcategory", "Revenue"])

            # Display the dataframe
            st.markdown("**📋 Summary Table - Revenue by Category & Subcategory**")
            st.dataframe(formatted_df, use_container_width=True)

        # --- Region Revenue ---
        # --- Region Revenue ---
        with tabs[1]:
            st.subheader("🌍 Revenue by Region")
            st.write("(Map-based visualization to identify regional performance.")
            region_data = df.groupby(["Region"]).agg({'Revenue': 'sum'}).reset_index()
            # Map region to dummy coordinates for illustration if real coordinates aren't present
            region_coords = {
                'North': [28.6139, 77.2090],
                'South': [12.9716, 77.5946],
                'East': [22.5726, 88.3639],
                'West': [19.0760, 72.8777],
                'Central': [23.2599, 77.4126]
            }
            region_data['lat'] = region_data['Region'].map(lambda x: region_coords[x][0])
            region_data['lon'] = region_data['Region'].map(lambda x: region_coords[x][1])
            fig = px.scatter_mapbox(region_data, lat="lat", lon="lon", size="Revenue", hover_name="Region",
                                    hover_data={"lat": False, "lon": False, "Revenue": True},
                                    color_discrete_sequence=["blue"], zoom=3, height=500)
            fig.update_layout(mapbox_style="open-street-map", margin={"r": 0, "t": 30, "l": 0, "b": 0})
            st.plotly_chart(fig, use_container_width=True)
            # 🧾 Summary Table
            st.markdown("**📋 Summary Table - Revenue by Region**")
            st.dataframe(region_data[["Region", "Revenue"]], use_container_width=True)
        # --- Channel Performance ---
        with tabs[2]:
            st.subheader("🌐 Online vs Offline Channel Revenue")
            st.write("This donut chart compares total revenue between online and offline sales channels.")
            channel_rev = df.groupby("Channel")["Revenue"].sum().reset_index()
            fig = px.pie(channel_rev, names='Channel', values='Revenue', hole=0.5,
                         color_discrete_sequence=px.colors.qualitative.Pastel)
            fig.update_traces(textinfo='percent+label')
            st.plotly_chart(fig, use_container_width=True)
            # 🧾 Summary Table
            st.markdown("**📋 Summary Table - Online vs Offline Revenue**")
            st.dataframe(channel_rev, use_container_width=True)

        # --- Promotion Impact ---

        with tabs[3]:
            st.subheader("🎯 Promotion vs Non-Promotion Revenue")
            st.write("This box plot shows the revenue distribution for transactions with and without promotions.")
            fig = px.box(df, x="Promotion_Applied", y="Revenue", color="Promotion_Applied",
                         title="Revenue Distribution Based on Promotion Application")
            st.plotly_chart(fig, use_container_width=True)
            # 🧾 Summary Table
            promo_table = df.groupby("Promotion_Applied")["Revenue"].describe().reset_index()
            st.markdown("**📋 Summary Table - Promotion vs Non-Promotion Revenue**")
            st.dataframe(promo_table, use_container_width=True)

        # --- Customer Type ---
        with tabs[4]:
            st.subheader("👥 Customer Type Revenue")
            cust_rev = df.groupby("Customer_Type")["Revenue"].sum().reset_index()
            fig = px.pie(cust_rev, names='Customer_Type', values='Revenue', hole=0.3,
                         color_discrete_sequence=px.colors.sequential.RdBu)
            st.plotly_chart(fig, use_container_width=True)
            # 🧾 Summary Table
            st.markdown("**📋 Summary Table - Revenue by Customer Type**")
            st.dataframe(cust_rev, use_container_width=True)

        # --- Seasonality ---
        with tabs[5]:
            st.subheader("☀️ Seasonal Revenue Trends")
            season_rev = df.groupby("Season")["Revenue"].sum().reset_index().sort_values("Revenue", ascending=False)
            fig = px.bar(season_rev, x='Season', y='Revenue', color='Season', text_auto=True)
            st.plotly_chart(fig, use_container_width=True)
            # 🧾 Summary Table
            st.markdown("**📋 Summary Table - Seasonal Revenue**")
            st.dataframe(season_rev, use_container_width=True)

        # --- Anomaly Trends ---
        with tabs[6]:
            st.subheader("⚠️ Anomaly Revenue Heatmap")
            st.write("This heatmap shows revenue anomalies over time by subcategory.")
            anomaly_df = df[df['Anomaly_Flag'] == 'Yes']
            heat_df = anomaly_df.copy()
            heat_df['Month'] = heat_df['Date'].dt.to_period('M').astype(str)
            pivot_table = heat_df.pivot_table(values='Revenue', index='Subcategory', columns='Month', aggfunc='sum',
                                              fill_value=0)
            fig = px.imshow(pivot_table,
                            labels=dict(x="Month", y="Subcategory", color="Revenue"),
                            aspect="auto",
                            color_continuous_scale='Reds',
                            title="Revenue Anomalies Heatmap by Subcategory and Month")
            st.plotly_chart(fig, use_container_width=True)
            # 🧾 Summary Table
            st.markdown("**📋 Summary Table - Monthly Revenue Anomalies (Subcategory-wise)**")
            st.dataframe(pivot_table, use_container_width=True)
#-----------------------------------------------------------------------------------------------------------------------------------


    elif selected == "Revenue Forecasting":
        st.markdown(
            f"""
            <style>
            .center {{
                display: flex;
                flex-direction: column;
                align-items: center;
                justify-content: flex-start;
            }}
            .hero {{
                font-size: 32px;
                font-weight: 700;
                margin-top: 10px;
                color: #1f77b4;
                text-align: center;
            }}
            .subtext {{
                 font-size: 18px;
                 font-weight: 400;
                 color: #CCCCCC;
                 text-align: center;
                 max-width: 900px;
                 line-height: 1.6;
                 font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
                 margin-bottom: 40px;
            }}
            </style>
            <div class="center">
                <div class="hero">AI-Powered Revenue Forecasting</div>
                <div class="subtext">
                    Predict future revenue with cutting-edge machine learning models.<br>
                    This module leverages historical trends, seasonality, and external factors to deliver accurate, data-driven forecasts.
                </div>
            </div>
            """,
            unsafe_allow_html=True
        )


        @st.cache_data
        def load_data():

            df = pd.read_csv("AI_revenue_forecasting_dataset.csv")

            df["Date"] = pd.to_datetime(df["Date"])

            return df


        df = load_data()

        # Label Encoding

        from sklearn.preprocessing import LabelEncoder

        categorical_columns = [col for col in df.select_dtypes(include='object').columns if col != "Date"]

        le = LabelEncoder()

        df[categorical_columns] = df[categorical_columns].apply(le.fit_transform)

        # Preprocess Daily Revenue

        df_daily = df.groupby("Date")["Revenue"].sum().reset_index().sort_values("Date")

        df_daily["Month"] = df_daily["Date"].dt.month

        df_daily["Day"] = df_daily["Date"].dt.day

        df_daily["Weekday"] = df_daily["Date"].dt.weekday

        df_daily["Lag_1"] = df_daily["Revenue"].shift(1)

        df_daily["Lag_7"] = df_daily["Revenue"].shift(7)

        df_daily = df_daily.dropna()

        # Select future forecast range

        forecast_days = st.selectbox("⏳ Forecast Period", [30, 60, 90], index=0)

        last_date = df_daily["Date"].max()

        future_start_date = st.date_input("📅 Select Forecast Start Date", value=last_date + timedelta(days=1),

                                          min_value=last_date + timedelta(days=1))
        # Calculate future last date by adding forecast days to the future start date
        future_last_date = future_start_date + timedelta(days=forecast_days - 1)

        # Show the future last date in the Streamlit app
        st.write(f"📅 Forecast will end on: {future_last_date}")

        # Train Model

        from sklearn.ensemble import RandomForestRegressor

        X = df_daily[["Month", "Day", "Weekday", "Lag_1", "Lag_7"]]

        y = df_daily["Revenue"]

        model = RandomForestRegressor(n_estimators=100, random_state=42)

        model.fit(X, y)

        # Forecast Future Dates

        future_dates = [future_start_date + timedelta(days=i) for i in range(forecast_days)]

        future_df = pd.DataFrame({"Date": future_dates})

        # Convert the 'Date' column to datetime if it's not already in datetime format
        future_df["Date"] = pd.to_datetime(future_df["Date"], errors='coerce')

        # Now you can safely use the .dt accessor
        future_df["Month"] = future_df["Date"].dt.month

        future_df["Day"] = future_df["Date"].dt.day

        future_df["Weekday"] = future_df["Date"].dt.weekday

        # Generate lag values from previous df_daily

        last_known = df_daily.tail(7).copy()

        all_dates = pd.concat([df_daily[["Date", "Revenue"]], future_df[["Date"]]], ignore_index=True)

        revenue_list = df_daily["Revenue"].tolist()

        for i in range(forecast_days):
            lag_1 = revenue_list[-1]

            lag_7 = revenue_list[-7] if len(revenue_list) >= 7 else lag_1

            row = {

                "Month": future_df.loc[i, "Month"],

                "Day": future_df.loc[i, "Day"],

                "Weekday": future_df.loc[i, "Weekday"],

                "Lag_1": lag_1,

                "Lag_7": lag_7,

            }

            pred = model.predict(pd.DataFrame([row]))[0]

            revenue_list.append(pred)

            future_df.loc[i, "Revenue"] = pred

            future_df.loc[i, "Lag_1"] = lag_1

            future_df.loc[i, "Lag_7"] = lag_7

        future_df["Flag"] = "Predicted"

        df_daily["Flag"] = "Actual"

        full_df = pd.concat([df_daily[["Date", "Revenue", "Flag"]], future_df[["Date", "Revenue", "Flag"]]],
                            ignore_index=True)

        # Plot actual vs predicted

        import plotly.express as px

        fig = px.line(

            full_df, x="Date", y="Revenue", color="Flag", markers=True,

            title="📊 Revenue Forecast: Actual vs Predicted", template="plotly_white"

        )

        st.plotly_chart(fig, use_container_width=True)

        # Show and download results


        # Convert 'Date' column to date only (remove time part)
        full_df["Date"] = full_df["Date"].dt.date

        # Show forecasted data
        st.markdown("### 📋 Forecasted Data")
        st.dataframe(full_df.tail(forecast_days))  # Only future data

        # Download button for CSV
        st.download_button(
            label="📥 Download Forecasted Data (CSV)",
            data=full_df.to_csv(index=False),
            file_name="full_revenue_forecast.csv",
            mime="text/csv"
        )

#------------------------------------------------------------------------------------------------------------------------------

    elif selected == "Anomaly Detection":

        st.markdown(
            f"""
            <style>
            .center {{
                display: flex;
                flex-direction: column;
                align-items: center;
                justify-content: flex-start;
            }}
            .hero {{
                font-size: 32px;
                font-weight: 700;
                margin-top: 10px;
                color: #1f77b4;
                text-align: center;
            }}
            .subtext {{
                 font-size: 18px;
                 font-weight: 400;
                 color: #CCCCCC;
                 text-align: center;
                 max-width: 900px;
                 line-height: 1.6;
                 font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
                 margin-bottom: 40px;
            }}
            </style>
            <div class="center">
                <div class="hero">Revenue Anomaly Detection</div>
                <div class="subtext">
                    Instantly detect unusual spikes or drops in revenue with smart anomaly detection.<br>
                    Stay ahead of risks by identifying data outliers and operational irregularities early on.
                </div>
            </div>
            """,
            unsafe_allow_html=True
        )


        # Load Data

        @st.cache_data
        def load_data():

            df = pd.read_csv("AI_revenue_forecasting_dataset.csv")

            df["Date"] = pd.to_datetime(df["Date"])

            return df


        df = load_data()

        # Group by Date and Calculate Z-score

        df_daily = df.groupby("Date")["Revenue"].sum().reset_index().sort_values("Date")

        mean_rev = df_daily["Revenue"].mean()

        std_rev = df_daily["Revenue"].std()

        df_daily["Z_score"] = (df_daily["Revenue"] - mean_rev) / std_rev

        # Slider for Z-score threshold

        z_thresh = st.slider("⚙️ Anomaly Sensitivity (Z-score Threshold)", 1.0, 4.0, 2.5, 0.1)

        # Label anomalies

        df_daily["Anomaly"] = df_daily["Z_score"].apply(lambda z: "Yes" if abs(z) > z_thresh else "No")

        anomaly_data = df_daily[df_daily["Anomaly"] == "Yes"]

        # Strip time only for the line graph (showing only the date)

        df_daily["Date"] = df_daily["Date"].dt.date

        anomaly_data["Date"] = anomaly_data["Date"].dt.date

        # Plot actual revenue with anomalies

        import plotly.express as px

        fig = px.line(

            df_daily, x="Date", y="Revenue", markers=True,

            title="📊 Revenue with Anomalies Detected", template="plotly_white"

        )

        # Add anomalies as red markers on the plot

        fig.add_scatter(

            x=anomaly_data["Date"], y=anomaly_data["Revenue"], mode='markers',

            marker=dict(color='red', size=12, symbol='x'),

            name="Anomalies"

        )

        st.plotly_chart(fig, use_container_width=True)

        # Show anomalies table (including the time)

        st.subheader("📋 Detected Anomalies")

        # Display anomalies with full datetime information in the table

        st.dataframe(anomaly_data[["Date", "Revenue", "Z_score"]].sort_values("Date"))

        # Download button for anomalies data

        st.download_button(

            label="📥 Download Anomaly Data (CSV)",

            data=anomaly_data.to_csv(index=False),

            file_name="revenue_anomalies.csv",

            mime="text/csv"

        )

        # Optional: Z-score explanation

        with st.expander("📘 What is Z-score?"):

            st.markdown("""

            - A Z-score measures how far a value is from the mean in units of standard deviation.

            - Formula: **Z = (X - Mean) / StdDev**

            - Values with |Z| greater than the selected threshold are flagged as anomalies.

            """)
        # Optional: Z-score explanation


#----------------------------------------------------------------------------------------------------------------------------------


    elif selected == "User Dashboard":
        import streamlit as st
        import pandas as pd
        import plotly.express as px

        # Title
        st.markdown(
            f"""
            <style>
            .center {{
                display: flex;
                flex-direction: column;
                align-items: center;
                justify-content: flex-start;
            }}
            .hero {{
                font-size: 32px;
                font-weight: 700;
                margin-top: 10px;
                color: #1f77b4;
                text-align: center;
            }}
            .subtext {{
                 font-size: 18px;
                 font-weight: 400;
                 color: #CCCCCC;
                 text-align: center;
                 max-width: 900px;
                 line-height: 1.6;
                 font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
                 margin-bottom: 40px;
            }}
            </style>
            <div class="center">
                <div class="hero">Dashboard Overview</div>
                <div class="subtext">
                    Upload your model predictions and evaluate accuracy with insightful metrics and visualizations.<br>
                    Analyze trends, detect errors, and improve model decisions using real performance data.
                </div>
            </div>
            """,
            unsafe_allow_html=True
        )


        df = pd.read_csv("AI_revenue_forecasting_dataset.csv")
        df.columns = df.columns.str.strip().str.lower().str.replace(" ", "_")

        # Convert date column
        df['date'] = pd.to_datetime(df['date'])
        df['month'] = df['date'].dt.to_period('M').astype(str)

        st.markdown(
            "<h4 style='font-size:20px; font-weight:600; color:#FFFFFF;'> Input Data for Analysis</h4>",
            unsafe_allow_html=True
        )
        st.dataframe(df.head())

        # Layout in 3 columns
        col1, col2, col3 = st.columns(3)

        # 1. Monthly Revenue Trend
        with col1:
            st.markdown(
                "<h4 style='font-size:20px; font-weight:600; color:#FFFFFF;'> Monthly Revenue Trend</h4>",
                unsafe_allow_html=True
            )
            monthly_revenue = df.groupby("month")["revenue"].sum().reset_index()
            fig1 = px.line(monthly_revenue, x="month", y="revenue", markers=True)
            st.plotly_chart(fig1, use_container_width=True)

        # 2. Revenue by Product Category
        with col2:
            st.markdown(
                "<h4 style='font-size:20px; font-weight:600; color:#FFFFFF;'>Revenue by Category</h4>",
                unsafe_allow_html=True
            )
            cat_rev = df.groupby("category")["revenue"].sum().reset_index().sort_values("revenue", ascending=False)
            fig2 = px.bar(cat_rev, x="category", y="revenue", text_auto=True, color="revenue",
                          color_continuous_scale="Blues")
            st.plotly_chart(fig2, use_container_width=True)

        # 3. Regional Revenue Distribution
        with col3:
            st.markdown(
                "<h4 style='font-size:20px; font-weight:600; color:#FFFFFF;'>Revenue by Region</h4>",
                unsafe_allow_html=True
            )
            region_rev = df.groupby("region")["revenue"].sum().reset_index()
            fig3 = px.pie(region_rev, names="region", values="revenue", hole=0.4)
            st.plotly_chart(fig3, use_container_width=True)

        st.markdown("---")
        col4, col5, col6 = st.columns(3)

        # 4. Payment Method Preference
        with col4:
            st.markdown(
                "<h4 style='font-size:20px; font-weight:600; color:#FFFFFF;'>Payment Method Preference</h4>",
                unsafe_allow_html=True
            )
            pay_counts = df['payment_method'].value_counts().reset_index()
            pay_counts.columns = ['payment_method', 'count']
            fig4 = px.bar(pay_counts, x="payment_method", y="count", color="count", text_auto=True,
                          color_continuous_scale="Oranges")
            st.plotly_chart(fig4, use_container_width=True)

        # 5. Top 10 Products by Revenue
        with col5:
            st.markdown(
                "<h4 style='font-size:20px; font-weight:600; color:#FFFFFF;'>Top 10 Best-Selling Products</h4>",
                unsafe_allow_html=True
            )
            top_products = df.groupby("product_name")["revenue"].sum().reset_index().sort_values(by="revenue",
                                                                                                 ascending=False).head(
                10)
            fig5 = px.bar(top_products, x="revenue", y="product_name", orientation='h', text_auto=True,
                          color="revenue", color_continuous_scale="Greens")
            st.plotly_chart(fig5, use_container_width=True)

        # 6. Anomaly Flags Over Time
        with col6:
            st.markdown(
                "<h4 style='font-size:20px; font-weight:600; color:#FFFFFF;'>Anomaly Flags Over Time</h4>",
                unsafe_allow_html=True
            )
            anomaly_counts = df[df["anomaly_flag"] == "Yes"].groupby("month").size().reset_index(name="anomalies")
            fig6 = px.bar(anomaly_counts, x="month", y="anomalies", color="anomalies", text_auto=True,
                          color_continuous_scale="Reds")
            st.plotly_chart(fig6, use_container_width=True)

#-------------------------------------------------------------------------------------------------------------------------------
    elif selected == "Help & Documentation":
        st.markdown(
            f"""
            <style>
            .center {{
                display: flex;
                flex-direction: column;
                align-items: center;
                justify-content: flex-start;
            }}
            .hero {{
                font-size: 32px;
                font-weight: 700;
                margin-top: 10px;
                color: #1f77b4;
                text-align: center;
            }}
            .subtext {{
                 font-size: 18px;
                 font-weight: 400;
                 color: #CCCCCC;
                 text-align: center;
                 max-width: 900px;
                 line-height: 1.6;
                 font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
                 margin-bottom: 40px;
            }}
            </style>
            <div class="center">
                <div class="hero">Help & Documentation</div>
                
            </div>
            """,
            unsafe_allow_html=True
        )



        st.markdown("---")

        st.subheader(" About the Project")
        st.write("""
        This AI-driven system is designed to forecast revenue trends using machine learning models and provide valuable insights 
        for business growth. The platform includes modules for secure login, data ingestion, preprocessing, trend analysis, 
        forecasting, and actionable recommendations.
        """)
        st.markdown("---")

        st.subheader("How to Use")
        st.markdown("""
        1. **Register/Login** – Start by registering with your email and selecting your role (Admin/Analyst/Manager). OTP verification is required for secure registration.
        2. **Data Ingestion & Preprocessing** – Upload your dataset (CSV format). The system cleans and prepares your data automatically.
        3. **Trend Analysis** – Visualize revenue trends and patterns over time using line and bar charts.
        4. **Forecasting** – Get future revenue predictions using Random Forest-based models.
        5. **Business Recommendations** – View insights to help you improve planning, reduce risk, and grow your business.
        6. **Download Reports** – Export forecasts and insights as CSV for further use.
        """)
        st.markdown("---")

        st.subheader("Supported File Format")
        st.write("- Only `.csv` files are supported for data upload.")
        st.markdown("---")

        st.subheader("FAQs")
        st.markdown("""
        - **What kind of data do I need to upload?**  
          Your dataset should include date/time and revenue/sales columns. Additional columns like category, region, or department can improve insights.

        - **Can I use this for other domains?**  
          Yes! While optimized for business revenue, this system can work for any time series forecasting (e.g., traffic, inventory, usage).

        - **Is the forecasting model customizable?**  
          Currently, we use Random Forest, but future updates will support LSTM and ARIMA models too.
        """)
        st.markdown("---")

        st.subheader("Need Help?")
        st.write(
            "For technical support or feature requests, please contact the project developer or visit our company site:")
        st.write("Company Website: [https://infolabz.in/](https://infolabz.in/)")


# ------------------------------------------------------------------------------------------------------------------------------------ ---------------------------------------------------------------------------------------------------------------------------------
    elif selected == "Settings":
        import streamlit as st
        import pandas as pd
        import hashlib  # For password hashing
        import os


        # Function to hash passwords
        def hash_password(password):
            return hashlib.sha256(password.encode()).hexdigest()


        # Load user database from CSV or create a default admin user
        if os.path.exists("user_database.csv"):
            user_database = pd.read_csv("user_database.csv").to_dict(orient="records")
        else:
            user_database = [{
                "username": "admin",
                "password": hash_password("admin123"),
                "name": "Admin",
                "department": "IT",
                "role": "Admin"
            }]
            pd.DataFrame(user_database).to_csv("user_database.csv", index=False)


        # Save user database to CSV
        def save_user_database():
            df = pd.DataFrame(user_database)
            df.to_csv("user_database.csv", index=False)


        # Initialize session state if not present
        if 'username' not in st.session_state:
            st.session_state.username = "admin"
        if 'department' not in st.session_state:
            st.session_state.department = "IT"
        if 'role' not in st.session_state:
            st.session_state.role = "Admin"

        # ---------- SETTINGS PAGE ----------
        st.markdown(
            """
            <style>
            .center {
                display: flex;
                flex-direction: column;
                align-items: center;
                justify-content: flex-start;
            }
            .hero {
                font-size: 32px;
                font-weight: 700;
                margin-top: 10px;
                color: #1f77b4;
                text-align: center;
            }
            .subtext {
                 font-size: 18px;
                 font-weight: 400;
                 color: #CCCCCC;
                 text-align: center;
                 max-width: 900px;
                 line-height: 1.6;
                 font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
                 margin-bottom: 40px;
            }
            </style>
            <div class="center">
                <div class="hero">Settings Overview</div>
                <div class="subtext">
                    Manage your user preferences, role access levels, and configuration options.<br>
                    Customize your experience, update profile information, and configure system behavior securely.
                </div>
            </div>
            """,
            unsafe_allow_html=True
        )

        # ---------- USER PROFILE MANAGEMENT ----------
        st.markdown("<h4 style='font-size:18px; font-weight:600; color:#1f77b4;'>User Profile Management</h4>",
                    unsafe_allow_html=True)

        with st.expander("Update Profile Information"):
            new_name = st.text_input("Full Name", placeholder="Enter your full name", value=st.session_state.username)
            new_role = st.selectbox("Role", ["Analyst", "Manager", "Admin"],
                                    index=["Analyst", "Manager", "Admin"].index(st.session_state.role))

            if st.button("Update Profile"):
                for user in user_database:
                    if user["username"] == st.session_state.username:
                        user["name"] = new_name
                        user["role"] = new_role
                        break

                save_user_database()
                st.session_state.username = new_name
                st.session_state.role = new_role
                st.success("Profile updated successfully.")

        # ---------- CHANGE PASSWORD ----------
        st.markdown("<h4 style='font-size:18px; font-weight:600; color:#1f77b4;'>Change Password</h4>",
                    unsafe_allow_html=True)

        with st.expander("Reset Your Password"):
            current_password = st.text_input("Current Password", type="password")
            new_password = st.text_input("New Password", type="password")
            confirm_password = st.text_input("Confirm New Password", type="password")

            if st.button("Change Password"):
                user = next((user for user in user_database if user["username"] == st.session_state.username), None)
                if user and user["password"] == hash_password(current_password):
                    if new_password != confirm_password:
                        st.error("New passwords do not match.")
                    elif len(new_password) < 6:
                        st.warning("Password must be at least 6 characters.")
                    else:
                        user["password"] = hash_password(new_password)
                        save_user_database()
                        st.success("Password changed successfully.")
                else:
                    st.error("Current password is incorrect.")


    #---------------------------------------------------------------------------------------------------------------------------------------------

    elif selected == "About Us":
        st.markdown(
            f"""
            <style>
            .left {{
                display: flex;
                flex-direction: column;
                align-items: center;
                justify-content: flex-start;
            }}
            .hero {{
                font-size: 32px;
                font-weight: 700;
                margin-top: 10px;
                color: #1f77b4;
                text-align: center;
            }}
            .subtext {{
                 font-size: 18px;
                 font-weight: 400;
                 color: #CCCCCC;
                 text-align: center;
                 max-width: 900px;
                 line-height: 1.6;
                 font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
                 margin-bottom: 40px;
            }}
            </style>
            <div class="left">
                <div class="hero">About Us</div>
                
            </div>
            """,
            unsafe_allow_html=True
        )
        st.markdown("---")


        # Adding a project description

        st.subheader("AI-Driven Revenue Forecasting and Trend Analysis for Business Growth")
        st.write("""
           Our project aims to leverage AI and machine learning techniques to forecast revenue trends for businesses across industries like retail, e-commerce, and SaaS. By using advanced algorithms, the system provides actionable insights to assist with financial planning, risk mitigation, and resource allocation.
           """)

        st.subheader("Our Mission")
        st.write(
            "To provide innovative AI-driven solutions for business growth through intelligent data analysis and forecasting.")

        st.subheader("Developer Information")
        st.write(
            "This project is developed by **Janki Panchal**, a Data Science Intern under the guidance of **Mr. Kirit Suthar**, an Software Developer at InfoLabz.")

        st.subheader("Contact Us")
        st.write("Email: jankipanchal1609@example.com")
        st.write("LinkedIn: www.linkedin.com/in/janki-panchal-jp1609")
        st.write("Our company website: https://infolabz.in/")

# ---------------------------------------------------------------------------------------------------------------------------

    elif selected == "Log Out":
        st.session_state.authenticated = False
        st.session_state.username = ""
        st.query_params.clear()
        st.success("Logged out successfully.")
        rerun()
