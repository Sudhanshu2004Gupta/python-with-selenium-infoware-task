# python-with-selenium-infoware-task
import time

import pandas as pd
from selenium import webdriver
from selenium.common.exceptions import TimeoutException, NoSuchElementException
from selenium.webdriver.common.by import By
from selenium.webdriver.support import expected_conditions as ec
from selenium.webdriver.support.ui import WebDriverWait

AMAZON_LOGIN_URL = "https://www.amazon.in/ap/signin"
BEST_SELLERS_URL = "https://www.amazon.in/gp/bestsellers/"
CATEGORIES = [
    "kitchen", "shoes", "computers", "electronics"
]  
OUTPUT_FILE = "amazon_best_sellers.csv"

driver = webdriver.Chrome() 
wait = WebDriverWait(driver, 10)


def login_amazon(email, password):
    """Logs into Amazon using the provided credentials."""
    driver.get(AMAZON_LOGIN_URL)
    try:
        email_input = wait.until(ec.presence_of_element_located((By.ID, "ap_email")))
        email_input.send_keys(email)
        driver.find_element(By.ID, "continue").click()

        password_input = wait.until(ec.presence_of_element_located((By.ID, "ap_password")))
        password_input.send_keys(password)
        driver.find_element(By.ID, "signInSubmit").click()
        print("Login successful")
    except TimeoutException:
        print("Login failed. Check credentials or page load time.")
        driver.quit()


def scrape_category(category_urls):
    """Scrapes data for a given category."""
    driver.get(category_urls)
    time.sleep(2)  # Allow page to load

    products = []
    try:
        for _ in range(30):  # Adjust for pagination to scrape up to 1500 products
            product_elements = driver.find_elements(By.CSS_SELECTOR, "div.zg-grid-general-face-out")
            for product in product_elements:
                try:
                    name = product.find_element(By.CSS_SELECTOR, ".p13n-sc-truncate-desktop-type2").text
                    price = product.find_element(By.CSS_SELECTOR, ".p13n-sc-price").text
                    discount = product.find_element(By.CSS_SELECTOR, ".a-row.a-size-small").text  # Approximation
                    rating = product.find_element(By.CSS_SELECTOR, ".a-icon-alt").text
                    sold_by = product.find_element(By.CSS_SELECTOR, ".a-size-small.a-color-secondary").text
                    ship_from = "Amazon"  # Assume as placeholder
                    images = product.find_elements(By.CSS_SELECTOR, ".zg-item img")

                    product_data = {
                        "Product Name": name,
                        "Price": price,
                        "Discount": discount,
                        "Rating": rating,
                        "Sold By": sold_by,
                        "Ship From": ship_from,
                        "Images": [img.get_attribute("src") for img in images],
                        "Category": category_urls.split("/")[-1]
                    }
                    products.append(product_data)
                except NoSuchElementException:
                    continue

            try:
                next_button = driver.find_element(By.CSS_SELECTOR, "li.a-last a")
                next_button.click()
                time.sleep(2)  # Allow time for the next page to load
            except NoSuchElementException:
                break  # No more pages
    except Exception as e:
        print(f"Error scraping category: {e}")

    return products


def save_data(data, file_name):
    """Saves the scraped data to a CSV file."""
    df = pd.DataFrame(data)
    df.to_csv(file_name, index=False)
    print(f"Data saved to {file_name}")


if __name__ == "__main__":
    try:
        EMAIL = "your_email@example.com"
        PASSWORD = "your_password"

        login_amazon(EMAIL, PASSWORD)

        all_data = []
        for category in CATEGORIES:
            print(f"Scraping category: {category}")
            category_url = f"{BEST_SELLERS_URL}{category}/ref=zg_bs_nav_{category}_0"
            category_data = scrape_category(category_url)
            all_data.extend(category_data)

        save_data(all_data, OUTPUT_FILE)
    finally:
        driver.quit()
