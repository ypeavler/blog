# **10 Essential Engineering Practices for Junior Developers**

## **Introduction**  
Great software engineering is not just about writing code that works—it's about writing code that is clean, maintainable, and scalable. The best engineers develop **good habits** that make their code easier to read, debug, and extend over time.  

For junior engineers (2-5 years of experience), adopting these best practices early will **save you countless hours of debugging and refactoring** later in your career. This post covers **10 essential engineering habits**, along with practical examples to help you apply them in real-world projects.  

---

## **1. Name Variables and Functions Properly**  
Poor naming conventions can make even simple code unreadable. Your goal should be to name variables and functions **descriptively** so that another developer (or future you) can understand their purpose immediately.  

### **Bad Example (Unclear Naming)**  
```python
def c(p, d):
    return p * d
```
What does `c` mean? What are `p` and `d`? This is unreadable.  

### **Better Example (Descriptive Naming)**  
```python
def calculate_total_price(price_per_item, quantity):
    return price_per_item * quantity
```
- Variables should describe **what they represent**, not just **how they are used**.  
- Function names should state **what they do**—use **verbs** for actions (`calculate_total_price`, `fetch_user_data`).  
- Stick to a consistent naming style (camelCase, snake_case) depending on your language.  

---

## **2. Develop Good Commit Habits**  
Using Git effectively is a must-have skill for any developer. **Messy commit histories make debugging painful, while good commit practices help track and understand changes.**  

### **Best Practices for Commits:**  
✅ Commit small, self-contained changes  
✅ Write meaningful commit messages  
✅ Use feature branches (`feature/signup-page`, `bugfix/login-crash`)  

### **Example of a Bad Commit Message:**  
❌ `"fixed some bugs"`  

### **Better Commit Message Format:**  
✅ `"Fix incorrect date formatting in user profile page"`  

💡 **Pro Tip:** A good commit message should **explain why the change was made, not just what was changed**.  

---

## **3. Good PR Habits**  

### **Best Practices for Pull Requests:**  
- Keep PRs **small and focused** (one feature or bug fix at a time).  
- Include a **clear description**:  
  - What does this PR do?  
  - How was it tested?  
- Review other people's PRs with constructive feedback.  

---

## **4. Follow the Single Responsibility Principle (SRP)**  
([Read more: Robert C. Martin on SRP](https://blog.cleancoder.com/uncle-bob/2014/05/08/SingleReponsibilityPrinciple.html))  
A function should do **one thing well**—if it's doing multiple things, it's a sign of **code smell**.  

### **Bad Example (Violates SRP - One Function Doing Too Much)**  
```python
def process_order(order):
    # Validate order
    if not order.items:
        raise ValueError("Order has no items")
    
    # Save order to database
    print(f"Saving order {order.id} to database")
    
    # Send confirmation email
    print(f"Sending confirmation email to {order.user_email}")
    
    return "Order processed"
```

### **Better Example (SRP Applied - Each Function Has One Job)**  
```python
def validate_order(order):
    """Check if order is valid."""
    if not order.items:
        raise ValueError("Order has no items")

def save_order_to_db(order):
    """Save order details to the database."""
    print(f"Saving order {order.id} to database")

def send_order_confirmation(order):
    """Send order confirmation email to user."""
    print(f"Sending confirmation email to {order.user_email}")

def process_order(order):
    """Process an order by validating, saving, and notifying the user."""
    validate_order(order)
    save_order_to_db(order)
    send_order_confirmation(order)
    return "Order processed"
```

💡 **Rule of Thumb:** If a function **does more than one thing**, **break it down into smaller functions**!  

---

## **5. Move All I/O to the Edges**  
When writing functions, keep **input/output (I/O) operations at the top level** and separate them from the core logic. This makes testing easier.  

### **Example:**  
```python
def process_user_request(request):
    user_data = fetch_user_data(request.user_id)  # I/O at the top
    return calculate_user_score(user_data)  # Pure function, easy to test
```
💡 **Why?**  
- **Core logic remains testable** (you can mock `fetch_user_data`).  
- **Easier to reason about the function’s behavior**.  

---

## **6. Write Good Tests (Test Pyramid)**  
([Read more: Martin Fowler on the Test Pyramid](https://martinfowler.com/bliki/TestPyramid.html))  
A solid test strategy follows the **Test Pyramid** model:

![Test Pyramid](https://martinfowler.com/bliki/images/testPyramid/testPyramid.png)

### **Test Pyramid Breakdown:**  
- **Unit Tests (Fast, 70-80%)** – Test individual functions; should run in milliseconds.
- **Integration Tests (Slower, 10-20%)** – Test how components interact.
- **End-to-End Tests (Slowest, ~10%)** – Test full workflows but should be limited.

💡 **Keep unit tests small and isolated, and ensure tests run fast to encourage frequent testing.**

---

## **9. Fail Fast & Handle Errors Gracefully**  
([Read more: Martin Fowler on Fail Fast](https://martinfowler.com/ieeeSoftware/failFast.pdf))  
### **Bad Example (Fails Silently, Hard to Debug)**  
```python
try:
    do_something()
except Exception:
    pass  # Silent failure! BAD
```

### **Better Example (Log the Error and Fail Loudly if Needed)**  
```python
try:
    do_something()
except Exception as e:
    logging.error(f"Something went wrong: {e}")
    raise  # Don't hide errors!
```

---

## **10. Don't Reinvent the Wheel (But Avoid Dependency Hell)**  
([Read more: NPM Docs on Dependency Management](https://docs.npmjs.com/about-packages-and-modules))  
Using third-party packages can save time, but too many dependencies can lead to **version conflicts and security issues**.  

### **Example of NPM Dependency Hell:**  
- `left-pad` breaking the internet because a tiny package was removed.  
- Having **50+ dependencies** for something simple like date formatting.  

### **Rule of Thumb:**  
✅ **Use libraries** for complex, standard solutions (e.g., authentication, database).  
❌ **Avoid unnecessary dependencies** for simple things you can write yourself.  

---

## **Conclusion**  
These **engineering practices will make your code cleaner and more maintainable**. Start small—**focus on improving 1% every day**. 🚀

