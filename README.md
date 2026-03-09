# SpecForge-project--Technical-Framework-or-a-Content-Management-Foundation
This is my first real project


"""
SpecForge - Car E-commerce Platform with Physics Simulation

A comprehensive car and parts marketplace featuring:
- User profiles and authentication
- Car and parts inventory management
- Shopping cart and wishlist functionality
- Sales analytics and reporting
- Stock notification system
- HotWheels car physics simulation
"""

from datetime import datetime, timedelta
from collections import defaultdict, Counter
from dataclasses import dataclass
from typing import Optional, Dict, List, Any, Tuple, Generator, Callable
import json
import random
import string
import os
import math
import hashlib
import uuid
from decimal import Decimal
from abc import ABC, abstractmethod
from functools import wraps


# ============================================================================
# CONSTANTS
# ============================================================================

# Validation constants
MIN_USERNAME_LENGTH = 3
MIN_PASSWORD_LENGTH = 6
VERIFICATION_CODE_LENGTH = 8

# Car simulation constants
DEFAULT_WHEELBASE = 2.5
MAX_STEERING_ANGLE = 45.0
STEERING_INCREMENT = 5.0
REVERSE_GEAR_SPEED_LIMIT = 20
GEAR_SPEED_MULTIPLIER = 30
ACCELERATION_POWER = 2.0
BRAKE_DECELERATION = 5.0
HANDBRAKE_SPEED_MULTIPLIER = 0.7
HANDBRAKE_STEERING_MULTIPLIER = 1.8
SPIN_OUT_THRESHOLD = 0.5
SPIN_OUT_SPEED_THRESHOLD = 50
SPIN_OUT_CHANCE = 0.1
SPIN_OUT_ANGLE = 30
SPEED_TO_RADIUS_CONVERSION = 0.1

# Stock management constants
DEFAULT_LOW_STOCK_THRESHOLD = 2

# Car categories
CAR_CATEGORIES = ["sedans", "sports", "SUVs", "trucks", "electric", "custom"]

# Date format
DATE_FORMAT = "%Y-%m-%d"

# Loyalty discount (DiscountEngine)
LOYALTY_SPEND_THRESHOLD = 50_000
LOYALTY_DISCOUNT_PERCENT = 0.10


# ============================================================================
# TAX AND CURRENCY LOCALIZATION (Decimal precision, no float for money)
# ============================================================================

class TaxProvider(ABC):
    """Abstract base for region-specific tax calculation (VAT, sales tax, etc.)."""
    
    @abstractmethod
    def compute_tax(self, amount: Decimal) -> Decimal:
        """
        Compute tax for a given subtotal. Never use float for money.
        
        Args:
            amount: Subtotal as Decimal
            
        Returns:
            Tax amount as Decimal
        """
        pass


class BulgariaVATTaxProvider(TaxProvider):
    """Bulgaria: 20% VAT (value-added tax)."""
    
    def __init__(self, rate: Decimal = Decimal("0.20")) -> None:
        self.rate = rate
    
    def compute_tax(self, amount: Decimal) -> Decimal:
        return (amount * self.rate).quantize(Decimal("0.01"))


class USASalesTaxProvider(TaxProvider):
    """USA: Sales tax (e.g. state rate; default illustrative rate)."""
    
    def __init__(self, rate: Decimal = Decimal("0.08")) -> None:
        self.rate = rate
    
    def compute_tax(self, amount: Decimal) -> Decimal:
        return (amount * self.rate).quantize(Decimal("0.01"))


class TaxAndCurrencyLocalizationEngine:
    """
    Handles decimal-precision money and region-specific tax.
    Uses Decimal throughout (never float for money) and a pluggable TaxProvider.
    """
    
    def __init__(self, tax_provider: TaxProvider, currency_code: str = "USD") -> None:
        """
        Args:
            tax_provider: Region implementation (Bulgaria VAT, USA Sales Tax, etc.)
            currency_code: e.g. "USD", "BGN", "EUR"
        """
        self.tax_provider = tax_provider
        self.currency_code = currency_code
    
    def subtotal_to_total(self, subtotal: Decimal) -> Decimal:
        """Compute total (subtotal + tax) with precise decimal math."""
        tax = self.tax_provider.compute_tax(subtotal)
        return (subtotal + tax).quantize(Decimal("0.01"))
    
    def compute_tax_only(self, subtotal: Decimal) -> Decimal:
        """Return just the tax for a subtotal."""
        return self.tax_provider.compute_tax(subtotal)
    
    def format_money(self, amount: Decimal) -> str:
        """Format amount for display (currency symbol/locale)."""
        if self.currency_code == "USD":
            return f"${amount:,.2f}"
        if self.currency_code == "BGN":
            return f"{amount:,.2f} лв."
        return f"{amount:,.2f} {self.currency_code}"


# ============================================================================
# DATA MODELS
# ============================================================================

@dataclass
class Product:
    """Represents a product that can be added to cart."""
    name: str
    price: float


class Car:
    """Type hint class for car attributes."""
    name: str
    brand: str
    model_year: int
    engine_type: str
    drivetrain: str
    color: str
    price: float
    stock: int
    drop_date: str
    features: List[str]


class Part:
    """Type hint class for part attributes."""
    part_type: str
    compatible_models: List[str]
    brand: str
    price: float
    stock: int
    performance_boost: Optional[float]


# ============================================================================
# USER MANAGEMENT
# ============================================================================


class Profile:
    """User profile with authentication and purchase tracking."""
    
    def __init__(self, username: str, email: str, password: str) -> None:
        """
        Initialize a user profile.
        
        Args:
            username: User's chosen username (min 3 chars)
            email: User's email address
            password: User's password (min 6 chars)
            
        Raises:
            ValueError: If username, email, or password validation fails
        """
        self.user_id = str(uuid.uuid4())
        self.username = self._validate_username(username)
        self.email = self._validate_email(email)
        self.password_hash = self._hash_password(password)
        self.date_joined = datetime.now()
        
        # User-related data
        self.cart: List[Product] = []
        self.wishlist: List[Dict[str, Any]] = []
        self.purchase_history: List[Dict[str, Any]] = []
        self.saved_builds: List[Dict[str, Any]] = []
        self.preferred_categories: List[str] = []
    
    def _validate_username(self, username: str) -> str:
        """Validate username meets minimum requirements."""
        if not username or len(username) < MIN_USERNAME_LENGTH:
            raise ValueError(f"Username must be at least {MIN_USERNAME_LENGTH} characters.")
        return username
    
    def _validate_email(self, email: str) -> str:
        """Validate email contains required characters."""
        if "@" not in email or "." not in email:
            raise ValueError("Invalid email format.")
        return email
    
    def _hash_password(self, password: str) -> str:
        """
        Hash password using SHA-256.
        
        Note: For production, use bcrypt or argon2 instead of SHA-256.
        """
        if len(password) < MIN_PASSWORD_LENGTH:
            raise ValueError(f"Password must be at least {MIN_PASSWORD_LENGTH} characters.")
        return hashlib.sha256(password.encode()).hexdigest()
    
    def update_profile(self, username: Optional[str] = None, 
                      password: Optional[str] = None) -> None:
        """
        Update profile information.
        
        Args:
            username: New username (optional)
            password: New password (optional)
        """
        if username:
            self.username = self._validate_username(username)
        if password:
            self.password_hash = self._hash_password(password)
    
    def generate_verification_code(self, length: int = VERIFICATION_CODE_LENGTH) -> str:
        """
        Generate a random verification code.
        
        Args:
            length: Length of the verification code
            
        Returns:
            Random alphanumeric string with punctuation
        """
        allowed_chars = string.ascii_letters + string.digits + string.punctuation
        return "".join(random.choice(allowed_chars) for _ in range(length))
    
    def get_user_stats(self) -> Dict[str, Any]:
        """
        Get comprehensive user statistics.
        
        Returns:
            Dictionary containing user statistics
        """
        total_orders = len(self.purchase_history)
        total_spent = sum(order.get("total", 0) for order in self.purchase_history)
        
        return {
            "user_id": self.user_id,
            "username": self.username,
            "days_on_platform": (datetime.now() - self.date_joined).days,
            "total_orders": total_orders,
            "total_spent": total_spent,
            "saved_builds_count": len(self.saved_builds),
            "preferred_categories": self.preferred_categories,
        }
    
    def __repr__(self) -> str:
        return f"Profile(username='{self.username}', email='{self.email}', user_id='{self.user_id}')"


# ============================================================================
# ANALYTICS
# ============================================================================

class SalesAnalytics:
    """Track and analyze sales data."""
    
    def __init__(self, journal: Optional['Journal'] = None) -> None:
        """
        Initialize analytics tracking.
        
        Args:
            journal: Optional event-sourcing journal to reconstruct state from events
        """
        self.order_history: List[Dict[str, Any]] = []
        self.category_sales: Dict[str, float] = defaultdict(float)
        self.total_sales: float = 0.0
        self.view_counts: Counter = Counter()
        self.user_activity: Dict[str, Dict[str, Any]] = defaultdict(
            lambda: {"orders": 0, "spent": 0.0, "abandoned_carts": 0}
        )
        self.journal = journal
    
    def log_order(self, user_id: str, items: List[Dict[str, Any]], total: float) -> None:
        """
        Log a completed order.
        
        Args:
            user_id: ID of the user who placed the order
            items: List of items in the order
            total: Total order value
        """
        order = {
            "user_id": user_id,
            "items": items,
            "total": total,
            "timestamp": datetime.now()
        }
        self.order_history.append(order)
        self.total_sales += total
        
        # Update category sales
        for item in items:
            category = item.get("category", "unknown")
            self.category_sales[category] += item.get("price", 0)
        
        # Update user activity
        self.user_activity[user_id]["orders"] += 1
        self.user_activity[user_id]["spent"] += total
    
    def log_view(self, item_name: str) -> None:
        """
        Log an item view.
        
        Args:
            item_name: Name of the viewed item
        """
        self.view_counts[item_name] += 1
    
    def get_revenue(self, period: str = "all") -> float:
        """
        Calculate revenue for a given period.
        
        Args:
            period: Time period ("all", "daily", "weekly", "monthly")
            
        Returns:
            Total revenue for the period
        """
        now = datetime.now()
        revenue = 0.0
        
        for order in self.order_history:
            timestamp = order["timestamp"]
            
            if period == "all":
                revenue += order["total"]
            elif period == "daily" and timestamp.date() == now.date():
                revenue += order["total"]
            elif period == "weekly" and timestamp >= now - timedelta(days=7):
                revenue += order["total"]
            elif period == "monthly" and timestamp >= now - timedelta(days=30):
                revenue += order["total"]
        
        return revenue
    
    def get_top_items(self, limit: int = 5) -> List[Tuple[str, int]]:
        """
        Get top selling items.
        
        Args:
            limit: Maximum number of items to return
            
        Returns:
            List of tuples (item_name, count) sorted by popularity
        """
        all_items = Counter()
        for order in self.order_history:
            for item in order["items"]:
                item_name = item.get("name", "unknown")
                all_items[item_name] += 1
        return all_items.most_common(limit)
    
    def get_conversion_rate(self) -> float:
        """
        Calculate conversion rate (% of users who made at least one purchase).
        
        Returns:
            Conversion rate as a percentage
        """
        total_users = len(self.user_activity)
        if total_users == 0:
            return 0.0
        
        buyers = sum(1 for u in self.user_activity.values() if u["orders"] > 0)
        return (buyers / total_users) * 100
    
    def get_user_activity(self, user_id: str) -> Dict[str, Any]:
        """
        Get activity statistics for a specific user.
        
        Args:
            user_id: User ID to look up
            
        Returns:
            Dictionary with user activity stats
        """
        return self.user_activity.get(
            user_id,
            {"orders": 0, "spent": 0.0, "abandoned_carts": 0}
        )
    
    def get_growth_report(self) -> Dict[str, Any]:
        """
        Generate a comprehensive growth report.

        Returns:
            Dictionary containing key metrics
        """
        return {
            "total_sales": self.total_sales,
            "category_sales": dict(self.category_sales),
            "most_viewed_items": self.view_counts.most_common(5),
            "total_orders": len(self.order_history),
            "conversion_rate": self.get_conversion_rate()
        }

    def qualifies_for_loyalty_discount(self, profile: 'Profile') -> bool:
        """
        Check if a profile has spent over the loyalty threshold ($50,000).
        Used by CartSystem to apply a 10% discount on their next purchase.

        Args:
            profile: User profile to check

        Returns:
            True if profile's total spent >= LOYALTY_SPEND_THRESHOLD
        """
        activity = self.user_activity.get(
            profile.user_id,
            {"orders": 0, "spent": 0.0, "abandoned_carts": 0}
        )
        return activity["spent"] >= LOYALTY_SPEND_THRESHOLD

    def get_state_from_events(self, up_to_index: Optional[int] = None) -> Optional[Dict[str, Any]]:
        """
        Reconstruct cart/orders state at a point in time by replaying journal events.
        Uses the event-sourcing Journal (generator replay) for data integrity.
        
        Args:
            up_to_index: Replay events only up to this index (None = all).
            
        Returns:
            {"cart", "orders", "total_sales"} from event replay, or None if no journal
        """
        if self.journal is None:
            return None
        return self.journal.reconstruct_state(up_to_index=up_to_index)


# ============================================================================
# EVENT SOURCING (LOG-BASED ARCHITECTURE)
# ============================================================================
# High-level banking and e-commerce backends use event sourcing for data integrity.
# Instead of storing only final state, we store every event and replay to reconstruct state.

EVENT_ITEM_ADDED = "ITEM_ADDED"
EVENT_ITEM_REMOVED = "ITEM_REMOVED"
EVENT_PRICE_CHANGED = "PRICE_CHANGED"
EVENT_CHECKOUT_STARTED = "CHECKOUT_STARTED"
EVENT_CHECKOUT_COMPLETED = "CHECKOUT_COMPLETED"


class Journal:
    """
    Event-sourcing journal: stores every cart/sales event and supports replay
    via a generator to reconstruct site state at any point in time.
    """
    
    def __init__(self) -> None:
        """Initialize empty event log."""
        self._events: List[Dict[str, Any]] = []
    
    def append(self, event_type: str, payload: Dict[str, Any]) -> None:
        """
        Record a single event.
        
        Args:
            event_type: One of ITEM_ADDED, ITEM_REMOVED, PRICE_CHANGED,
                        CHECKOUT_STARTED, CHECKOUT_COMPLETED
            payload: Event data (e.g. name, price, user_id, items)
        """
        self._events.append({
            "type": event_type,
            "payload": payload,
            "timestamp": datetime.now(),
            "index": len(self._events),
        })
    
    def replay(self, up_to_index: Optional[int] = None) -> Generator[Dict[str, Any], None, None]:
        """
        Generator that yields events in order (optionally up to a given index).
        Used to reconstruct state by applying events one by one.
        
        Args:
            up_to_index: If set, yield only events with index <= this value.
            
        Yields:
            Each event dict (type, payload, timestamp, index)
        """
        for evt in self._events:
            if up_to_index is not None and evt["index"] > up_to_index:
                return
            yield evt
    
    def reconstruct_state(self, up_to_index: Optional[int] = None) -> Dict[str, Any]:
        """
        Replay all events (up to index) and reconstruct cart + orders state.
        
        Returns:
            {"cart": [items], "orders": [orders], "total_sales": float}
        """
        cart: List[Dict[str, Any]] = []
        orders: List[Dict[str, Any]] = []
        total_sales = 0.0
        
        for evt in self.replay(up_to_index):
            etype = evt["type"]
            payload = evt["payload"]
            
            if etype == EVENT_ITEM_ADDED:
                cart.append({"name": payload.get("name"), "price": payload.get("price", 0)})
            elif etype == EVENT_ITEM_REMOVED:
                name = payload.get("name")
                cart = [i for i in cart if i.get("name") != name]
            elif etype == EVENT_PRICE_CHANGED:
                name = payload.get("name")
                for item in cart:
                    if item.get("name") == name:
                        item["price"] = payload.get("price", item["price"])
                        break
            elif etype == EVENT_CHECKOUT_STARTED:
                pass  # optional: track "in checkout" state
            elif etype == EVENT_CHECKOUT_COMPLETED:
                total = payload.get("total", 0)
                total_sales += total
                orders.append({
                    "user_id": payload.get("user_id"),
                    "total": total,
                    "timestamp": evt.get("timestamp"),
                })
                cart = []
        
        return {"cart": cart, "orders": orders, "total_sales": total_sales}


# ============================================================================
# INVENTORY MANAGEMENT
# ============================================================================

class CarInventoryManager:
    """Manage car inventory across categories."""
    
    def __init__(self) -> None:
        """Initialize inventory with empty categories."""
        self.cars_categories: Dict[str, List[Dict[str, Any]]] = {
            category: [] for category in CAR_CATEGORIES
        }
        self.car_brands: List[str] = [
            "Toyota", "Honda", "Ford", "Chevrolet", "BMW", "Mercedes-Benz",
            "Audi", "Volkswagen", "Hyundai", "Kia", "Nissan", "Mazda",
            "Subaru", "Lexus", "Dodge", "Jeep", "Ram", "Tesla", "Porsche",
            "Ferrari", "Lamborghini", "Bugatti", "McLaren", "Bentley",
            "Rolls-Royce", "Aston Martin", "Alfa Romeo", "Jaguar",
            "Peugeot", "Renault", "Skoda", "Volvo", "Citroën", "Fiat",
            "Mini", "Acura", "Infiniti", "Genesis", "Suzuki", "Mitsubishi"
        ]
    
    def add_car(self, name: str, category: str, brand: str, model_year: int,
                engine_type: str, drivetrain: str, color: str, price: float,
                stock: int, drop_date: str, features: List[str]) -> bool:
        """
        Add a car to inventory.
        
        Args:
            name: Car name/identifier
            category: Car category (must be valid)
            brand: Car manufacturer
            model_year: Year of manufacture
            engine_type: Engine description
            drivetrain: Drivetrain type (AWD, RWD, FWD)
            color: Car color
            price: Car price
            stock: Available stock quantity
            drop_date: Release date (YYYY-MM-DD format)
            features: List of feature tags
            
        Returns:
            True if car was added, False otherwise
        """
        if category not in self.cars_categories:
            print(f"❌ Invalid category: '{category}'")
            return False
        
        car = {
            "name": name,
            "brand": brand,
            "model_year": model_year,
            "engine_type": engine_type,
            "drivetrain": drivetrain,
            "color": color,
            "price": price,
            "stock": stock,
            "drop_date": drop_date,
            "features": features
        }
        
        self.cars_categories[category].append(car)
        return True
    
    def display_cars(self) -> None:
        """Display all cars in inventory organized by category."""
        print(f"\n🧾 SpecForge Inventory:")
        for category, items in self.cars_categories.items():
            if items:
                print(f"\n📦 {category.title()}:")
                for car in items:
                    print(
                        f"- {car['name']} ({car['brand']} {car['model_year']}) | "
                        f"{car['engine_type']} | {car['drivetrain']} | "
                        f"${car['price']} | Drop: {car['drop_date']} | "
                        f"Stock: {car['stock']}"
                    )
    
    def search_cars_fuzzy(self, query: str) -> List[Dict[str, Any]]:
        """
        Fuzzy search: return all cars whose name or brand starts with or
        contains the query (e.g. "Toyo" returns all Toyota cars).
        
        Args:
            query: Search string (case-insensitive)
            
        Returns:
            List of matching car dictionaries (may be empty)
        """

        query_lower = query.lower().strip()
        if not query_lower:
            return []
        matches: List[Dict[str, Any]] = []
        for items in self.cars_categories.values():
            for car in items:
                name_lower = car["name"].lower()
                brand_lower = car["brand"].lower()
                if (name_lower.startswith(query_lower) or query_lower in name_lower
                        or brand_lower.startswith(query_lower) or query_lower in brand_lower):
                    matches.append(car)
        return matches

    def search_car(self, name: str) -> Optional[Dict[str, Any]]:
        """
        Search for a car by name. Uses exact match first; if not found,
        uses fuzzy matching (startswith / in) and returns the first match.
        E.g. "Toyo" can match "Toyota Camry".
        
        Args:
            name: Car name or partial string to search for
            
        Returns:
            Car dictionary if found, None otherwise
        """
        name_lower = name.lower().strip()
        for items in self.cars_categories.values():
            for car in items:
                if car["name"].lower() == name_lower:
                    return car
        fuzzy = self.search_cars_fuzzy(name)
        if not fuzzy:
            print("❌ Car not found.")
            return None
        if len(fuzzy) > 1:
            print(f"🔍 {len(fuzzy)} matches found; returning first: {fuzzy[0]['name']}")
        return fuzzy[0]
    
    def remove_car(self, name: str) -> bool:
        """
        Remove a car from inventory.
        
        Args:
            name: Car name to remove
            
        Returns:
            True if car was removed, False otherwise
        """
        name_lower = name.lower()
        for category in self.cars_categories:
            for car in self.cars_categories[category]:
                if car['name'].lower() == name_lower:
                    self.cars_categories[category].remove(car)
                    return True
        print("❌ Car not found.")
        return False
    
    def upcoming_drops(self, days: int = 7) -> List[Dict[str, Any]]:
        """
        Get cars with upcoming drop dates.
        
        Args:
            days: Number of days to look ahead
            
        Returns:
            List of cars with upcoming drops
        """
        today = datetime.today().date()
        upcoming_drops = []
        
        for cars in self.cars_categories.values():
            for car in cars:
                try:
                    drop_date = datetime.strptime(car['drop_date'], DATE_FORMAT).date()
                    days_until_drop = (drop_date - today).days
                    if 0 < days_until_drop <= days:
                        upcoming_drops.append(car)
                except ValueError:
                    # Skip invalid date formats
                    continue
        
        return upcoming_drops


class PartInventoryManager:
    """Manage automotive parts inventory."""
    
    def __init__(self) -> None:
        """Initialize empty parts inventory."""
        self.parts: List[Dict[str, Any]] = []
    
    def add_part(self, part_type: str, compatible_models: List[str],
                brand: str, price: float, stock: int,
                performance_boost: Optional[float] = None) -> None:
        """
        Add a part to inventory.
        
        Args:
            part_type: Type of part (e.g., "brakes", "wheels")
            compatible_models: List of car models this part fits
            brand: Part manufacturer
            price: Part price
            stock: Available stock quantity
            performance_boost: Optional performance improvement percentage
        """
        part = {
            "part_type": part_type,
            "compatible_models": [model.lower() for model in compatible_models],
            "brand": brand,
            "price": price,
            "stock": stock,
            "performance_boost": performance_boost
        }
        
        self.parts.append(part)
        print(f"🔧 Part '{part_type}' added from {brand}.")
    
    def display_parts(self) -> None:
        """Display all parts in inventory."""
        print(f"\n🧩 Parts Inventory:")
        if not self.parts:
            print("No parts in inventory.")
            return
        
        for part in self.parts:
            boost_text = ""
            if part['performance_boost']:
                boost_text = f" | Boost: {part['performance_boost']}%"
            print(
                f"- {part['part_type']} ({part['brand']}) | "
                f"${part['price']} | Stock: {part['stock']}{boost_text}"
            )
    
    def find_compatible(self, car_name: str) -> List[Dict[str, Any]]:
        """
        Find parts compatible with a specific car.
        
        Args:
            car_name: Name of the car
            
        Returns:
            List of compatible parts
        """
        car_name_lower = car_name.lower()
        matches = [
            part for part in self.parts
            if car_name_lower in part["compatible_models"]
        ]
        if not matches:
            print(f"⚠️ No compatible parts found for {car_name}.")
        return matches
    
    def search_part(self, part_type: str, brand: str) -> Optional[Dict[str, Any]]:
        """
        Search for a specific part by type and brand.
        
        Args:
            part_type: Type of part to search for
            brand: Brand of part to search for
            
        Returns:
            Part dictionary if found, None otherwise
        """
        part_type_lower = part_type.lower()
        brand_lower = brand.lower()
        
        for part in self.parts:
            if (part['part_type'].lower() == part_type_lower and
                part['brand'].lower() == brand_lower):
                return part
        
        print(f"❌ Part '{brand} {part_type}' not found.")
        return None
    
    def remove_part(self, part_type: str, brand: str) -> bool:
        """
        Remove a part from inventory.
        
        Args:
            part_type: Type of part to remove
            brand: Brand of part to remove
            
        Returns:
            True if part was removed, False otherwise
        """
        part = self.search_part(part_type, brand)
        if part:
            self.parts.remove(part)
            print(f"❌ Removed {brand} {part_type}")
            return True
        return False


# ============================================================================
# SHOPPING FUNCTIONALITY
# ============================================================================

class SessionLogger:
    """
    Logger that writes session history to a .txt file with timestamps.
    Used to record cart additions and checkouts instead of only print().
    """
    
    def __init__(self, filepath: str = "specforge_session.txt") -> None:
        """
        Initialize logger with output file path.

        Args:
            filepath: Path to the .txt log file (created/appended)
        """
        self.filepath = filepath
    
    def log(self, message: str) -> None:
        """
        Write a line to the log file with a timestamp.

        Args:
            message: Message to record (e.g. "Car added: ...", "Checkout completed: ...")
        """
        timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        line = f"[{timestamp}] {message}\n"
        try:
            with open(self.filepath, "a", encoding="utf-8") as f:
                f.write(line)
        except IOError as e:
            print(f"❌ Logger error: {e}")


class CartSystem:
    """Shopping cart system for managing items before checkout."""
    
    def __init__(self, logger: Optional[SessionLogger] = None,
                 journal: Optional['Journal'] = None) -> None:
        """
        Initialize empty cart.
        
        Args:
            logger: Optional session logger for file I/O history
            journal: Optional event-sourcing journal for ITEM_ADDED / CHECKOUT_* events
        """
        self.items: List[Product] = []
        self.logger = logger
        self.journal = journal
    
    def add(self, product: Product) -> None:
        """
        Add a product to the cart.
        
        Args:
            product: Product to add
        """
        self.items.append(product)
        if self.journal:
            self.journal.append(EVENT_ITEM_ADDED, {"name": product.name, "price": product.price})
        print(f"🛒 Added: {product.name} (${product.price})")
        if self.logger:
            self.logger.log(f"Car added: {product.name} (${product.price})")
    
    def remove(self, product: Product) -> None:
        """
        Remove a product from the cart.
        
        Args:
            product: Product to remove
        """
        for index, item in enumerate(self.items):
            if item.name == product.name:
                del self.items[index]
                if self.journal:
                    self.journal.append(EVENT_ITEM_REMOVED, {"name": product.name})
                print(f"❌ Removed: {product.name}")
                return
        print(f"⚠️ The item '{product.name}' is not in your cart")
    
    def display(self, profile: Optional[Profile] = None,
                sales_analytics: Optional[SalesAnalytics] = None) -> None:
        """Display all items in the cart with total (with optional loyalty discount)."""
        print(f"\n🧾 Your cart:")
        
        if not self.items:
            print("🛒 Cart is empty.")
            return
        
        for item in self.items:
            print(f"- {item.name} (${item.price})")
        print(f"💰 Total: ${self.total(profile, sales_analytics):.2f}")
    
    def total(self, profile: Optional[Profile] = None,
              sales_analytics: Optional[SalesAnalytics] = None) -> float:
        """
        Calculate total cart value. Applies 10% loyalty discount if profile
        has spent over $50,000 (via sales_analytics).

        Args:
            profile: Optional user profile for loyalty discount
            sales_analytics: Optional analytics to check loyalty

        Returns:
            Sum of item prices, or 90% of that if loyalty discount applies
        """
        raw_total = sum(item.price for item in self.items)
        if profile and sales_analytics and sales_analytics.qualifies_for_loyalty_discount(profile):
            return raw_total * (1.0 - LOYALTY_DISCOUNT_PERCENT)
        return raw_total
    
    def checkout(self, profile: Optional[Profile] = None,
                 sales_analytics: Optional[SalesAnalytics] = None) -> bool:
        """
        Process cart checkout. Applies 10% loyalty discount if eligible.

        Args:
            profile: Optional user profile for loyalty discount
            sales_analytics: Optional analytics to check loyalty

        Returns:
            True if checkout successful, False if cart is empty
        """
        if not self.items:
            print(f"❌ Cart is empty.")
            return False
        
        total_amount = self.total(profile, sales_analytics)
        if self.journal:
            items_payload = [{"name": p.name, "price": p.price} for p in self.items]
            self.journal.append(EVENT_CHECKOUT_STARTED, {"items": items_payload})
        if profile and sales_analytics and sales_analytics.qualifies_for_loyalty_discount(profile):
            print(f"🎉 Loyalty discount applied (10% off)!")
        print(f"💳 Processing payment of ${total_amount:.2f}...")
        print("✅ Payment complete.")
        if self.journal:
            self.journal.append(EVENT_CHECKOUT_COMPLETED, {
                "user_id": profile.user_id if profile else None,
                "total": total_amount,
                "items": [{"name": p.name, "price": p.price} for p in self.items],
            })
        if self.logger:
            self.logger.log(f"Checkout completed: ${total_amount:.2f}")
        self.items.clear()
        return True


class Wishlist:
    """User wishlist for saving cars of interest."""
    
    def __init__(self, inventory: CarInventoryManager) -> None:
        """
        Initialize wishlist with inventory reference.
        
        Args:
            inventory: Car inventory manager
        """
        self.inventory = inventory
        self.wishlist: List[Dict[str, Any]] = []
    
    def add_to_wishlist(self, name: str) -> None:
        """
        Add a car to the wishlist.
        
        Args:
            name: Name of the car to add
        """
        car = self.inventory.search_car(name)
        if car:
            self.wishlist.append(car)
            print(f"💖 Added '{name}' to wishlist.")
    
    def remove_from_wishlist(self, name: str) -> None:
        """
        Remove a car from the wishlist.
        
        Args:
            name: Name of the car to remove
        """
        for car in self.wishlist:
            if car['name'] == name:
                self.wishlist.remove(car)
                print("❌ Removed from wishlist.")
                return
        print("❌ Car not in wishlist.")
    
    def display(self) -> None:
        """Display all cars in the wishlist."""
        print(f"\n📋 Wishlist:")
        if not self.wishlist:
            print("Wishlist is empty.")
            return
        
        for car in self.wishlist:
            print(f"- {car['name']} (${car['price']})")


# ============================================================================
# NOTIFICATION SYSTEM
# ============================================================================

class OutOfStockNotifier:
    """Notify users when out-of-stock items become available."""
    
    def __init__(self, inventory: CarInventoryManager) -> None:
        """
        Initialize notifier with inventory reference.
        
        Args:
            inventory: Car inventory manager
        """
        self.inventory = inventory
        self.subscribers: List[Dict[str, Any]] = []
    
    def subscribe(self, user_id: str, car_name: str, auto_add: bool = False) -> None:
        """
        Subscribe a user to notifications for a car.
        
        Args:
            user_id: ID of the subscribing user
            car_name: Name of the car to subscribe to
            auto_add: Whether to automatically add to cart when restocked
        """
        self.subscribers.append({
            "user_id": user_id,
            "car_name": car_name.lower(),
            "auto_add": auto_add
        })
        print(f"🔔 Subscribed: {user_id} → {car_name} (Auto-add: {auto_add})")
    
    def notify(self, car_name: str, updated_stock: int,
               wishlist: Wishlist, cart: CartSystem) -> None:
        """
        Notify subscribers when a car is restocked.
        
        Args:
            car_name: Name of the restocked car
            updated_stock: New stock quantity
            wishlist: User's wishlist (for auto-add feature)
            cart: User's cart (for auto-add feature)
        """
        if updated_stock < 0:
            return
        
        car_name_lower = car_name.lower()
        
        # Find and notify subscribers
        matching_subscribers = [
            sub for sub in self.subscribers
            if sub['car_name'] == car_name_lower
        ]
        
        for subscriber in matching_subscribers:
            print(f"📢 Notify: User {subscriber['user_id']} — '{car_name}' restocked!")
            
            if subscriber['auto_add']:
                car = self.inventory.search_car(car_name)
                if car:
                    cart.add(Product(name=car["name"], price=car["price"]))
        
        # Remove processed subscriptions
        self.subscribers = [
            sub for sub in self.subscribers
            if sub['car_name'] != car_name_lower
        ]


class PartTracker:
    """Track part stock levels and notify subscribers."""
    
    def __init__(self, parts: PartInventoryManager) -> None:
        """
        Initialize part tracker.
        
        Args:
            parts: Part inventory manager
        """
        self.parts = parts
        self.low_stock_threshold = DEFAULT_LOW_STOCK_THRESHOLD
        self.subscribers: List[Dict[str, Any]] = []
    
    def set_threshold(self, threshold: int) -> None:
        """
        Set the low stock threshold.
        
        Args:
            threshold: Stock level that triggers low stock warning
        """
        self.low_stock_threshold = threshold
        print(f"📉 Low stock threshold set to {threshold} units.")
    
    def subscribe(self, user_id: str, part_type: str, brand: str,
                  auto_add: bool = False) -> None:
        """
        Subscribe a user to notifications for a part.
        
        Args:
            user_id: ID of the subscribing user
            part_type: Type of part to subscribe to
            brand: Brand of part to subscribe to
            auto_add: Whether to automatically add to cart when restocked
        """
        self.subscribers.append({
            "user_id": user_id,
            "part_type": part_type.lower(),
            "brand": brand.lower(),
            "auto_add": auto_add
        })
        print(f"🔔 User {user_id} subscribed for {brand} {part_type} (Auto-add: {auto_add})")
    
    def update_stock(self, part_type: str, brand: str, new_stock: int,
                    cart: Optional[CartSystem] = None) -> None:
        """
        Update stock for a part and notify subscribers if restocked.
        
        Args:
            part_type: Type of part to update
            brand: Brand of part to update
            new_stock: New stock quantity
            cart: Optional cart for auto-add feature
        """
        part = self.parts.search_part(part_type, brand)
        if not part:
            return
        
        old_stock = part['stock']
        part['stock'] = max(0, new_stock)  # Prevent negative stock
        
        # Notify subscribers if restocked (was 0, now > 0)
        if old_stock == 0 and new_stock > 0:
            part_type_lower = part_type.lower()
            brand_lower = brand.lower()
            
            matching_subscribers = [
                sub for sub in self.subscribers
                if sub['part_type'] == part_type_lower and sub['brand'] == brand_lower
            ]
            
            for subscriber in matching_subscribers:
                print(f"📢 Notify: User {subscriber['user_id']} — {brand} {part_type} is back in stock!")
                
                if subscriber['auto_add'] and cart:
                    cart.add(Product(name=f"{brand} {part_type}", price=part["price"]))
            
            # Remove processed subscriptions
            self.subscribers = [
                sub for sub in self.subscribers
                if not (sub['part_type'] == part_type_lower and sub['brand'] == brand_lower)
            ]
        
        # Check for low stock
        if new_stock < self.low_stock_threshold:
            print(f"⚠️ Warning: {brand} {part_type} stock is low ({new_stock})")
    
    def display_subscribers(self) -> None:
        """Display all active part subscriptions."""
        print(f"\n📋 Part Subscriptions:")
        if not self.subscribers:
            print("No active subscriptions.")
            return
        
        for subscriber in self.subscribers:
            print(
                f"- User {subscriber['user_id']} → "
                f"{subscriber['brand'].title()} {subscriber['part_type']} "
                f"(Auto-add: {subscriber['auto_add']})"
            )


# ============================================================================
# UPDATE SYSTEM
# ============================================================================

class Updater:
    """Update car and part attributes."""
    
    def __init__(self, inventory: CarInventoryManager,
                 part_inventory: PartInventoryManager) -> None:
        """
        Initialize updater with inventory references.
        
        Args:
            inventory: Car inventory manager
            part_inventory: Part inventory manager
        """
        self.inventory = inventory
        self.part_inventory = part_inventory
    
    def update_car(self, name: str, **kwargs: Any) -> bool:
        """
        Update car attributes.
        
        Args:
            name: Name of the car to update
            **kwargs: Attribute-value pairs to update
            
        Returns:
            True if update successful, False otherwise
        """
        car = self.inventory.search_car(name)
        if not car:
            return False
        
        for attr_name, value in kwargs.items():
            if value is not None and attr_name in car:
                car[attr_name] = value
        
        print(f"🔧 Updated: {name}")
        return True
    
    def update_part(self, part_type: str, brand: str, **kwargs: Any) -> bool:
        """
        Update part attributes.
        
        Args:
            part_type: Type of part to update
            brand: Brand of part to update
            **kwargs: Attribute-value pairs to update
            
        Returns:
            True if update successful, False otherwise
        """
        part = self.part_inventory.search_part(part_type, brand)
        if not part:
            return False
        
        for attr_name, value in kwargs.items():
            if value is not None and attr_name in part:
                part[attr_name] = value
        
        print(f"🔧 Updated part: {brand} {part_type}")
        return True


# ============================================================================
# FILTERING SYSTEM
# ============================================================================

class ItemFilter:
    """Filter and sort items based on criteria."""
    
    def __init__(self, items: List[Dict[str, Any]],
                 filters: Dict[str, Any]) -> None:
        """
        Initialize filter with items and filter criteria.
        
        Args:
            items: List of items to filter
            filters: Dictionary of filter criteria
        """
        self.items = items
        self.filters = filters
        self.filtered: List[Dict[str, Any]] = []
    
    def apply(self) -> 'ItemFilter':
        """
        Apply filters to items.
        
        Returns:
            Self for method chaining
        """
        self.filtered = [
            item for item in self.items
            if all(
                item.get(name) == value
                for name, value in self.filters.items()
                if name != "sort"
            )
        ]
        return self
    
    def sort(self) -> 'ItemFilter':
        """
        Sort filtered items.
        
        Returns:
            Self for method chaining
        """
        sort_key = self.filters.get('sort')
        
        if sort_key == "price_ascending":
            self.filtered.sort(key=lambda item: item['price'])
        elif sort_key == "price_descending":
            self.filtered.sort(key=lambda item: item['price'], reverse=True)
        elif sort_key == "drop_date_ascending":
            self.filtered.sort(key=lambda item: item['drop_date'])
        elif sort_key == "drop_date_descending":
            self.filtered.sort(key=lambda item: item['drop_date'], reverse=True)
        elif sort_key == 'name_ascending':
            self.filtered.sort(key=lambda x: x['name'])
        elif sort_key == 'name_descending':
            self.filtered.sort(key=lambda x: x['name'], reverse=True)
        
        return self
    
    def results(self) -> List[Dict[str, Any]]:
        """
        Get filtered and sorted results.
        
        Returns:
            List of filtered items
        """
        return self.filtered


# ============================================================================
# STORAGE SYSTEM & MULTI-LEVEL CACHING
# ============================================================================
# Write-back cache: RAM (dict) first, sync to disk (JSON) on flush to avoid disk I/O on every read.

class CacheManager:
    """
    Write-back cache: data lives in RAM (dictionary); syncs to disk (JSON) only
    when flush_to_disk() is called. Reads hit RAM first; disk is read only on cache miss.
    """
    
    def __init__(self, file_path: str) -> None:
        self._file_path = file_path
        self._ram_cache: Optional[Dict[str, List[Dict[str, Any]]]] = None
        self._dirty = False
    
    def ensure_loaded(self) -> None:
        """Load from JSON into RAM only if not already cached (avoids disk I/O on every read)."""
        if self._ram_cache is not None:
            return
        try:
            if os.path.exists(self._file_path):
                with open(self._file_path, "r", encoding="utf-8") as f:
                    self._ram_cache = json.load(f)
            else:
                self._ram_cache = {cat: [] for cat in CAR_CATEGORIES}
            self._dirty = False
        except (IOError, json.JSONDecodeError) as e:
            print(f"❌ Cache load error: {e}")
            self._ram_cache = {cat: [] for cat in CAR_CATEGORIES}
    
    def get_categories(self) -> Dict[str, List[Dict[str, Any]]]:
        """Return the in-memory categories (load from disk if needed)."""
        self.ensure_loaded()
        return self._ram_cache  # type: ignore
    
    def update_categories(self, data: Dict[str, List[Dict[str, Any]]]) -> None:
        """Update cache from inventory and mark dirty for write-back."""
        self._ram_cache = {k: list(v) for k, v in data.items()}
        self._dirty = True
    
    def flush_to_disk(self) -> None:
        """Write-back: persist RAM cache to JSON."""
        self.ensure_loaded()
        if not self._dirty:
            return
        try:
            with open(self._file_path, "w", encoding="utf-8") as f:
                json.dump(self._ram_cache, f, indent=4, default=str)
            self._dirty = False
        except IOError as e:
            print(f"❌ Cache flush error: {e}")


def cache_manager(attr_name: str = "_cache") -> Callable[..., Any]:
    """
    Decorator for inventory search: ensure cache is loaded (RAM) before calling;
    if data is in the dictionary return from cache; otherwise load from JSON then use cache.
    """
    def decorator(func: Callable[..., Any]) -> Callable[..., Any]:
        @wraps(func)
        def wrapper(self: Any, *args: Any, **kwargs: Any) -> Any:
            cache: CacheManager = getattr(self, attr_name)
            cache.ensure_loaded()
            # Sync inventory from cache so search runs on cached data
            self.inventory.cars_categories = cache.get_categories()
            return func(self, *args, **kwargs)
        return wrapper
    return decorator


class CarStorage(Updater):
    """Manage persistent storage for car inventory with write-back RAM cache."""
    
    FILE_PATH = "cars_storage.json"
    
    def __init__(self, inventory: CarInventoryManager,
                 part_inventory: PartInventoryManager) -> None:
        """
        Initialize storage system with multi-level cache (RAM + JSON write-back).
        
        Args:
            inventory: Car inventory manager
            part_inventory: Part inventory manager
        """
        super().__init__(inventory, part_inventory)
        self.inventory = inventory
        self._cache = CacheManager(self.FILE_PATH)
        
        if not os.path.exists(self.FILE_PATH):
            self._create_storage_file()
        
        self.load_cars()
    
    def _create_storage_file(self) -> None:
        """Create initial storage file with empty categories."""
        initial_data = {category: [] for category in CAR_CATEGORIES}
        try:
            with open(self.FILE_PATH, "w", encoding="utf-8") as f:
                json.dump(initial_data, f, indent=4)
        except IOError as e:
            print(f"❌ Error creating storage file: {e}")
    
    def load_cars(self) -> None:
        """Load cars from cache (RAM); only hits disk if cache is cold)."""
        self._cache.ensure_loaded()
        self.inventory.cars_categories = self._cache.get_categories()
        print("📂 Cars loaded from storage.")
    
    def save_cars(self) -> None:
        """Write-back: persist current inventory to cache then flush to disk."""
        self._cache.update_categories(self.inventory.cars_categories)
        self._cache.flush_to_disk()
        print("💾 Cars saved to storage.")
    
    @cache_manager("_cache")
    def search_car_cached(self, name: str) -> Optional[Dict[str, Any]]:
        """Search for a car by name using cache (RAM first; disk only on cache miss)."""
        return self.inventory.search_car(name)
    
    def add_and_store_car(self, **car_data: Any) -> None:
        """
        Add car via manager and save to storage.
        
        Args:
            **car_data: Car data to add
        """
        self.inventory.add_car(**car_data)
        self.save_cars()
    
    def remove_and_store_car(self, name: str) -> None:
        """
        Remove car via manager and save to storage.
        
        Args:
            name: Name of car to remove
        """
        self.inventory.remove_car(name)
        self.save_cars()
    
    def update_and_store_car(self, name: str, **kwargs: Any) -> None:
        """
        Update car attributes and save to storage.
        
        Args:
            name: Name of car to update
            **kwargs: Attribute-value pairs to update
        """
        self.update_car(name, **kwargs)
        self.save_cars()
    
    def upcoming_drops(self, days: int = 7) -> List[Dict[str, Any]]:
        """
        Check upcoming drops from storage.
        
        Args:
            days: Number of days to look ahead
            
        Returns:
            List of cars with upcoming drops
        """
        return self.inventory.upcoming_drops(days)


# ============================================================================
# PHYSICS SIMULATION
# ============================================================================

class HotWheelsCar:
    """HotWheels car physics simulation with realistic handling."""
    
    SURFACE_EFFECTS = {
        "asphalt": {"sport": 1.0, "offroad": 0.7, "wet": 0.8},
        "dirt": {"sport": 0.5, "offroad": 1.0, "wet": 0.6},
        "wet": {"sport": 0.4, "offroad": 0.6, "wet": 1.0}
    }
    
    def __init__(self, name: str, drivetrain: str = "RWD",
                 tires: str = "sport", surface: str = "asphalt") -> None:
        """
        Initialize car with physics properties.
        
        Args:
            name: Car identifier
            drivetrain: Drivetrain type (RWD, FWD, AWD)
            tires: Tire type (sport, offroad, wet)
            surface: Current surface (asphalt, dirt, wet)
        """
        self.name = name
        self.x: float = 0.0
        self.y: float = 0.0
        self.direction: float = 0.0  # degrees
        self.steering_angle: float = 0.0
        self.speed: float = 0.0
        self.gear: int = 0  # -1=R, 0=N, 1–6
        self.max_gears = 6
        self.handbrake_on: bool = False
        self.drivetrain = drivetrain
        self.tires = tires
        self.surface = surface
        self.wheelbase = DEFAULT_WHEELBASE
    
    def grip(self) -> float:
        """
        Calculate grip coefficient based on surface and tires.
        
        Returns:
            Grip coefficient (0.0 to 1.0)
        """
        return self.SURFACE_EFFECTS[self.surface][self.tires]
    
    def accelerate(self) -> None:
        """Accelerate the car based on current gear and grip."""
        if self.gear == 0:  # Neutral
            return
        
        gear_limit = (REVERSE_GEAR_SPEED_LIMIT if self.gear == -1
                     else self.gear * GEAR_SPEED_MULTIPLIER)
        
        if abs(self.speed) < gear_limit:
            power = ACCELERATION_POWER * self.grip()
            if self.gear == -1:  # Reverse
                self.speed -= power
            else:
                self.speed += power
    
    def brake(self) -> None:
        """Apply brakes based on grip."""
        decel = BRAKE_DECELERATION * self.grip()
        if self.speed > 0:
            self.speed = max(0, self.speed - decel)
        elif self.speed < 0:
            self.speed = min(0, self.speed + decel)
    
    def handbrake(self) -> None:
        """Engage handbrake (reduces speed, increases steering angle)."""
        self.handbrake_on = True
        self.speed *= HANDBRAKE_SPEED_MULTIPLIER
        self.steering_angle *= HANDBRAKE_STEERING_MULTIPLIER
    
    def release_handbrake(self) -> None:
        """Release handbrake (restores normal steering)."""
        self.handbrake_on = False
        self.steering_angle /= HANDBRAKE_STEERING_MULTIPLIER
    
    def shift_up(self) -> None:
        """Shift to a higher gear."""
        if self.gear < self.max_gears:
            self.gear += 1
    
    def shift_down(self) -> None:
        """Shift to a lower gear (or reverse)."""
        if self.gear > -1:
            self.gear -= 1
    
    def steer_left(self) -> None:
        """Steer left."""
        self.steering_angle = max(-MAX_STEERING_ANGLE,
                                  self.steering_angle - STEERING_INCREMENT)
    
    def steer_right(self) -> None:
        """Steer right."""
        self.steering_angle = min(MAX_STEERING_ANGLE,
                                  self.steering_angle + STEERING_INCREMENT)
    
    def update_position(self) -> None:
        """Update car position based on physics simulation."""
        if self.speed == 0:
            return
        
        grip = self.grip()
        
        # Handle turning based on steering angle and grip
        if self.steering_angle != 0:
            effective_angle = self.steering_angle * grip
            if effective_angle != 0:
                radius = self.wheelbase / math.sin(math.radians(abs(effective_angle)))
                turning_rate = (self.speed / radius) * (180 / math.pi)
                
                if effective_angle < 0:
                    self.direction -= turning_rate
                else:
                    self.direction += turning_rate
        
        # Drivetrain-specific handling characteristics
        if abs(self.speed) > 40:
            understeer_oversteer_factor = (1 - grip) * 2
            if self.drivetrain == "FWD":
                self.direction -= understeer_oversteer_factor  # FWD understeers
            elif self.drivetrain == "RWD":
                self.direction += understeer_oversteer_factor  # RWD oversteers
        
        # Random spin-out chance at low grip and high speed
        if (grip < SPIN_OUT_THRESHOLD and abs(self.speed) > SPIN_OUT_SPEED_THRESHOLD
                and random.random() < SPIN_OUT_CHANCE):
            spin_angle = random.choice([-SPIN_OUT_ANGLE, SPIN_OUT_ANGLE])
            print(f"💥 {self.name} spins out! ({spin_angle}°)")
            self.direction += spin_angle
        
        # Normalize direction to 0-360 degrees
        self.direction %= 360
        
        # Update position based on direction and speed
        radians = math.radians(self.direction)
        self.x += math.cos(radians) * self.speed * SPEED_TO_RADIUS_CONVERSION
        self.y += math.sin(radians) * self.speed * SPEED_TO_RADIUS_CONVERSION
    
    def status(self) -> str:
        """
        Get current car status as formatted string.
        
        Returns:
            Status string with position, direction, speed, etc.
        """
        gear_text = "R" if self.gear == -1 else "N" if self.gear == 0 else str(self.gear)
        return (
            f"{self.name}: Pos=({self.x:.1f},{self.y:.1f}) "
            f"Dir={self.direction:.1f}° Speed={self.speed:.1f} "
            f"Gear={gear_text} Steering={self.steering_angle:.1f}° "
            f"Surface={self.surface} Grip={self.grip():.2f}"
        )


class CommandProcessor:
    """Process commands for controlling HotWheelsCar."""
    
    def __init__(self, car: HotWheelsCar) -> None:
        """
        Initialize command processor with a car.
        
        Args:
            car: HotWheelsCar instance to control
        """
        self.car = car
    
    def process(self, command: str) -> None:
        """
        Process a control command.
        
        Args:
            command: Command string (w/s/a/d for movement, etc.)
        """
        command_lower = command.lower()
        
        if command_lower == "w":
            self.car.accelerate()
        elif command_lower == "s":
            self.car.brake()
        elif command_lower == "a":
            self.car.steer_left()
        elif command_lower == "d":
            self.car.steer_right()
        elif command == " ":
            self.car.handbrake()
        elif command == "release":
            self.car.release_handbrake()
        elif command == "LEFT_CLICK":
            self.car.shift_up()
        elif command.startswith("surface:"):
            new_surface = command.split(":")[1]
            if new_surface in self.car.SURFACE_EFFECTS:
                self.car.surface = new_surface
                print(f"🌍 Surface changed to {self.car.surface}")
            else:
                print(f"⚠️ Invalid surface: {new_surface}")
        
        self.car.update_position()


# ============================================================================
# MAIN EXECUTION
# ============================================================================

def main() -> None:
    """Main execution function demonstrating system functionality."""
    # Initialize systems (event-sourcing journal shared by analytics and cart)
    car_inventory = CarInventoryManager()
    part_inventory = PartInventoryManager()
    journal = Journal()
    analytics = SalesAnalytics(journal=journal)
    session_logger = SessionLogger("specforge_session.txt")
    cart = CartSystem(logger=session_logger, journal=journal)
    
    # Create sample users
    users: List[Profile] = []
    for i in range(10):
        username = f"User{i}{random.choice(string.ascii_letters)}"
        email = f"user{i}@example.com"
        password = f"Pass{i}123!"
        try:
            users.append(Profile(username, email, password))
        except ValueError as e:
            print(f"❌ Error creating user {i}: {e}")
            continue
    
    # Add sample cars
    brands = ["Toyota", "Ferrari", "Tesla", "BMW", "Ford", "Chevrolet", "Lamborghini"]
    engine_types = ["V6", "V8", "Electric", "Hybrid"]
    drivetrains = ["RWD", "FWD", "AWD"]
    colors = ["Red", "Blue", "Black", "White", "Yellow", "Green"]
    features_pool = ["turbo", "AWD", "eco", "self-driving", "sport", "hybrid", "custom kit"]
    
    for i in range(15):
        name = f"Car{i}"
        category = random.choice(CAR_CATEGORIES)
        brand = random.choice(brands)
        model_year = random.randint(2015, 2025)
        engine_type = random.choice(engine_types)
        drivetrain = random.choice(drivetrains)
        color = random.choice(colors)
        price = random.randint(20000, 350000)
        stock = random.randint(0, 5)
        drop_date = (datetime.now() + timedelta(days=random.randint(1, 30))).strftime(DATE_FORMAT)
        features = random.sample(features_pool, 2)
        
        car_inventory.add_car(
            name, category, brand, model_year, engine_type,
            drivetrain, color, price, stock, drop_date, features
        )
    
    # Add sample parts
    part_types = ["brakes", "wheels", "turbo", "suspension", "spoiler", "engine chip"]
    for i in range(20):
        part_type = random.choice(part_types)
        compatible_models = [
            f"Car{random.randint(0, 14)}",
            f"Car{random.randint(0, 14)}"
        ]
        brand = random.choice(brands)
        price = random.randint(500, 10000)
        stock = random.randint(0, 5)
        performance_boost = random.choice([None, random.randint(5, 50)])
        
        part_inventory.add_part(
            part_type, compatible_models, brand, price, stock, performance_boost
        )
    
    # Users add cars to wishlist and parts to cart
    wishlists: List[Wishlist] = []
    for user in users:
        wishlist = Wishlist(car_inventory)
        for _ in range(random.randint(1, 3)):
            car_name = f"Car{random.randint(0, 14)}"
            wishlist.add_to_wishlist(car_name)
        wishlists.append(wishlist)
        
        # Add random parts to cart
        for _ in range(random.randint(0, 2)):
            part = random.choice(part_inventory.parts)
            cart.add(Product(
                name=f"{part['brand']} {part['part_type']}",
                price=part['price']
            ))
    
    # Complete some purchases (pass profile + analytics for loyalty discount & logging)
    # Give first user prior spend so they qualify for 10% loyalty discount on next purchase
    analytics.log_order(
        users[0].user_id,
        [{"name": "Prior purchase", "price": 60_000, "category": "sedans"}],
        60_000.0
    )
    for user in users[:5]:
        if cart.items:
            total = cart.total(profile=user, sales_analytics=analytics)
            order_items = [
                {
                    "name": item.name,
                    "price": item.price,
                    "category": random.choice(CAR_CATEGORIES)
                }
                for item in cart.items
            ]
            analytics.log_order(user.user_id, order_items, total)
            cart.checkout(profile=user, sales_analytics=analytics)
    
    # HotWheelsCar simulation with random commands
    surfaces = ["asphalt", "dirt", "wet"]
    for idx, user in enumerate(users):
        hw_car = HotWheelsCar(
            f"HWCar{idx}",
            drivetrain=random.choice(drivetrains),
            tires=random.choice(["sport", "offroad", "wet"]),
            surface=random.choice(surfaces)
        )
        cmd_processor = CommandProcessor(hw_car)
        
        commands = random.choices(
            ["w", "s", "a", "d", " ", "release", "LEFT_CLICK",
             "surface:asphalt", "surface:dirt", "surface:wet"],
            k=10
        )
        
        for command in commands:
            cmd_processor.process(command)
            print(hw_car.status())
    
    # Part tracker subscriptions and stock updates
    part_tracker = PartTracker(part_inventory)
    for user in users:
        part = random.choice(part_inventory.parts)
        part_tracker.subscribe(
            user.user_id,
            part["part_type"],
            part["brand"],
            auto_add=random.choice([True, False])
        )
    
    for part in part_inventory.parts:
        new_stock = random.randint(0, 5)
        part_tracker.update_stock(
            part["part_type"],
            part["brand"],
            new_stock,
            cart=cart
        )
    
    # Test car updates
    updater = Updater(car_inventory, part_inventory)
    for i in range(5):
        car_name = f"Car{random.randint(0, 14)}"
        updater.update_car(
            car_name,
            price=random.randint(30000, 360000),
            color=random.choice(["Gold", "Silver", "Black"])
        )
    
    # Display final inventory and analytics
    car_inventory.display_cars()
    part_inventory.display_parts()
    print("\n📊 Analytics Growth Report:", analytics.get_growth_report())

    # Event-sourcing: reconstruct state from journal (replay events)
    state = analytics.get_state_from_events()
    if state:
        print("\n📜 Event-sourcing state (replayed from journal):",
              "orders=", len(state.get("orders", [])), "total_sales=", state.get("total_sales", 0))

    # Tax/currency localization (Decimal, no float for money)
    usa_tax = TaxAndCurrencyLocalizationEngine(USASalesTaxProvider(), "USD")
    bulgaria_tax = TaxAndCurrencyLocalizationEngine(BulgariaVATTaxProvider(), "BGN")
    subtotal = Decimal("1000.00")
    print("\n💰 Tax (Decimal): USA total:", usa_tax.format_money(usa_tax.subtotal_to_total(subtotal)),
          "| Bulgaria total:", bulgaria_tax.format_money(bulgaria_tax.subtotal_to_total(subtotal)))


if __name__ == "__main__":
    main()


