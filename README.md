# CafeManagementSystem
import javax.swing.*;
import java.awt.*;
import java.awt.event.*;
import java.io.*;
import java.util.*;
import java.util.List;

public class CafeManagementSystem extends JFrame {
    // Data classes
    static class User implements Serializable {
        String username, password, role;
        User(String u, String p, String r) { username = u; password = p; role = r; }
    }
    
    static class MenuItem implements Serializable {
        String name; double price;
        MenuItem(String n, double p) { name = n; price = p; }
        public String toString() { return name + " - $" + String.format("%.2f", price); }
    }
    
    static class OrderItem implements Serializable {
        MenuItem item; int quantity;
        OrderItem(MenuItem i, int q) { item = i; quantity = q; }
        double getTotal() { return item.price * quantity; }
        public String toString() { 
            return item.name + " x" + quantity + " = $" + String.format("%.2f", getTotal()); 
        }
    }
    
    static class Order implements Serializable {
        static int nextId = 1;
        int id; String customerName; List<OrderItem> items = new ArrayList<>();
        boolean paid = false;
        Order(String c) { id = nextId++; customerName = c; }
        double getTotal() {
            return items.stream().mapToDouble(OrderItem::getTotal).sum();
        }
        public String toString() {
            return "Order #" + id + " - " + customerName + " - $" + 
                   String.format("%.2f", getTotal()) + (paid ? " [PAID]" : " [PENDING]");
        }
    }

    // Data storage
    private List<User> users = new ArrayList<>();
    private List<MenuItem> menu = new ArrayList<>();
    private List<Order> orders = new ArrayList<>();

    // File names
    private final String USERS_FILE = "users.dat";
    private final String MENU_FILE = "menu.dat";
    private final String ORDERS_FILE = "orders.dat";

    // GUI Components
    private CardLayout cardLayout = new CardLayout();
    private JPanel mainPanel = new JPanel(cardLayout);

    // Login components
    private JTextField usernameField = new JTextField(15);
    private JPasswordField passwordField = new JPasswordField(15);

    // Menu management components
    private DefaultListModel<MenuItem> menuListModel = new DefaultListModel<>();
    private JList<MenuItem> menuList = new JList<>(menuListModel);

    // Order management components
    private DefaultListModel<Order> orderListModel = new DefaultListModel<>();
    private JList<Order> orderList = new JList<>(orderListModel);

    private User currentUser = null;

    public CafeManagementSystem() {
        loadData();

        if (users.isEmpty()) {
            users.add(new User("admin", "admin", "Admin"));
            saveData();
        }

        setTitle("Cafe Management System");
        setSize(700, 500);
        setLocationRelativeTo(null);
        setDefaultCloseOperation(EXIT_ON_CLOSE);

        initLoginPanel();
        initMenuPanel();
        initOrderPanel();

        add(mainPanel);

        cardLayout.show(mainPanel, "login");
    }

    private void initLoginPanel() {
        JPanel loginPanel = new JPanel(new GridBagLayout());
        GridBagConstraints gbc = new GridBagConstraints();
        gbc.insets = new Insets(5,5,5,5);

        gbc.gridx = 0; gbc.gridy = 0;
        loginPanel.add(new JLabel("Username:"), gbc);
        gbc.gridx = 1;
        loginPanel.add(usernameField, gbc);

        gbc.gridx = 0; gbc.gridy = 1;
        loginPanel.add(new JLabel("Password:"), gbc);
        gbc.gridx = 1;
        loginPanel.add(passwordField, gbc);

        JButton loginBtn = new JButton("Login");
        gbc.gridx = 0; gbc.gridy = 2; gbc.gridwidth = 2;
        loginPanel.add(loginBtn, gbc);

        loginBtn.addActionListener(e -> login());

        mainPanel.add(loginPanel, "login");
    }

    private void initMenuPanel() {
        JPanel panel = new JPanel(new BorderLayout());

        menuList.setSelectionMode(ListSelectionModel.SINGLE_SELECTION);
        JScrollPane menuScroll = new JScrollPane(menuList);

        JPanel btnPanel = new JPanel();

        JButton addItemBtn = new JButton("Add Menu Item");
        JButton deleteItemBtn = new JButton("Delete Selected Item");
        JButton logoutBtn = new JButton("Logout");
        JButton goOrderBtn = new JButton("Go to Orders");

        btnPanel.add(addItemBtn);
        btnPanel.add(deleteItemBtn);
        btnPanel.add(goOrderBtn);
        btnPanel.add(logoutBtn);

        addItemBtn.addActionListener(e -> addMenuItem());
        deleteItemBtn.addActionListener(e -> deleteMenuItem());
        logoutBtn.addActionListener(e -> logout());
        goOrderBtn.addActionListener(e -> {
            refreshOrderList();
            cardLayout.show(mainPanel, "orders");
        });

        panel.add(new JLabel("Menu Management", SwingConstants.CENTER), BorderLayout.NORTH);
        panel.add(menuScroll, BorderLayout.CENTER);
        panel.add(btnPanel, BorderLayout.SOUTH);

        mainPanel.add(panel, "menu");
    }

    private void initOrderPanel() {
        JPanel panel = new JPanel(new BorderLayout());

        orderList.setSelectionMode(ListSelectionModel.SINGLE_SELECTION);
        JScrollPane orderScroll = new JScrollPane(orderList);

        JPanel btnPanel = new JPanel();

        JButton newOrderBtn = new JButton("Create New Order");
        JButton payOrderBtn = new JButton("Mark as Paid");
        JButton backToMenuBtn = new JButton("Back to Menu");
        JButton logoutBtn = new JButton("Logout");

        btnPanel.add(newOrderBtn);
        btnPanel.add(payOrderBtn);
        btnPanel.add(backToMenuBtn);
        btnPanel.add(logoutBtn);

        newOrderBtn.addActionListener(e -> createOrder());
        payOrderBtn.addActionListener(e -> payOrder());
        backToMenuBtn.addActionListener(e -> {
            refreshMenuList();
            cardLayout.show(mainPanel, "menu");
        });
        logoutBtn.addActionListener(e -> logout());

        panel.add(new JLabel("Order Management", SwingConstants.CENTER), BorderLayout.NORTH);
        panel.add(orderScroll, BorderLayout.CENTER);
        panel.add(btnPanel, BorderLayout.SOUTH);

        mainPanel.add(panel, "orders");
    }

    private void login() {
        String username = usernameField.getText().trim();
        String password = new String(passwordField.getPassword());

        for (User u : users) {
            if (u.username.equals(username) && u.password.equals(password)) {
                currentUser = u;
                JOptionPane.showMessageDialog(this, "Welcome " + u.username + "!");
                refreshMenuList();
                cardLayout.show(mainPanel, "menu");
                usernameField.setText("");
                passwordField.setText("");
                return;
            }
        }
        JOptionPane.showMessageDialog(this, "Invalid username or password.", "Login Failed", JOptionPane.ERROR_MESSAGE);
    }

    private void logout() {
        currentUser = null;
        saveData();
        cardLayout.show(mainPanel, "login");
    }

    private void addMenuItem() {
        String name = JOptionPane.showInputDialog(this, "Enter item name:");
        if (name == null || name.trim().isEmpty()) return;

        String priceStr = JOptionPane.showInputDialog(this, "Enter item price:");
        if (priceStr == null || priceStr.trim().isEmpty()) return;

        try {
            double price = Double.parseDouble(priceStr);
            menu.add(new MenuItem(name.trim(), price));
            refreshMenuList();
            saveData();
        } catch (NumberFormatException e) {
            JOptionPane.showMessageDialog(this, "Invalid price entered.", "Error", JOptionPane.ERROR_MESSAGE);
        }
    }

    private void deleteMenuItem() {
        MenuItem selected = menuList.getSelectedValue();
        if (selected != null) {
            menu.remove(selected);
            refreshMenuList();
            saveData();
        } else {
            JOptionPane.showMessageDialog(this, "Please select a menu item to delete.", "No Selection", JOptionPane.WARNING_MESSAGE);
        }
    }

    private void refreshMenuList() {
        menuListModel.clear();
        for (MenuItem m : menu) menuListModel.addElement(m);
    }

    private void createOrder() {
        String customerName = JOptionPane.showInputDialog(this, "Enter customer name:");
        if (customerName == null || customerName.trim().isEmpty()) return;

        Order order = new Order(customerName.trim());

        while (true) {
            MenuItem selectedItem = selectMenuItemDialog();
            if (selectedItem == null) break;

            String qtyStr = JOptionPane.showInputDialog(this, "Enter quantity for " + selectedItem.name + ":");
            if (qtyStr == null) break;

            try {
                int qty = Integer.parseInt(qtyStr);
                if (qty <= 0) throw new NumberFormatException();
                order.items.add(new OrderItem(selectedItem, qty));
            } catch (NumberFormatException e) {
                JOptionPane.showMessageDialog(this, "Invalid quantity entered.", "Error", JOptionPane.ERROR_MESSAGE);
            }

            int more = JOptionPane.showConfirmDialog(this, "Add more items?", "Continue", JOptionPane.YES_NO_OPTION);
            if (more != JOptionPane.YES_OPTION) break;
        }

        if (!order.items.isEmpty()) {
            orders.add(order);
            refreshOrderList();
            saveData();
            JOptionPane.showMessageDialog(this, "Order created! Total: $" + String.format("%.2f", order.getTotal()));
        } else {
            JOptionPane.showMessageDialog(this, "No items added. Order cancelled.");
        }
    }

    private MenuItem selectMenuItemDialog() {
        if (menu.isEmpty()) {
            JOptionPane.showMessageDialog(this, "Menu is empty. Please add items first.", "Menu Empty", JOptionPane.WARNING_MESSAGE);
            return null;
        }
        MenuItem[] itemsArray = menu.toArray(new MenuItem[0]);
        MenuItem selected = (MenuItem) JOptionPane.showInputDialog(this, "Select Menu Item:",
                "Menu", JOptionPane.PLAIN_MESSAGE, null, itemsArray, itemsArray[0]);
        return selected;
    }

    private void refreshOrderList() {
        orderListModel.clear();
        for (Order o : orders) orderListModel.addElement(o);
    }

    private void payOrder() {
        Order selected = orderList.getSelectedValue();
        if (selected == null) {
            JOptionPane.showMessageDialog(this, "Select an order to mark as paid.", "No Selection", JOptionPane.WARNING_MESSAGE);
            return;
        }
        if (selected.paid) {
            JOptionPane.showMessageDialog(this, "Order is already paid.", "Info", JOptionPane.INFORMATION_MESSAGE);
            return;
        }
        int confirm = JOptionPane.showConfirmDialog(this, 
                "Mark order #" + selected.id + " as paid?", 
                "Confirm Payment", JOptionPane.YES_NO_OPTION);
        if (confirm == JOptionPane.YES_OPTION) {
            selected.paid = true;
            refreshOrderList();
            saveData();
        }
    }

    private void loadData() {
        users = loadList(USERS_FILE);
        menu = loadList(MENU_FILE);
        orders = loadList(ORDERS_FILE);

        // Update next order id
        int maxId = orders.stream().mapToInt(o -> o.id).max().orElse(0);
        if (maxId >= Order.nextId) Order.nextId = maxId + 1;
    }

    private void saveData() {
        saveList(users, USERS_FILE);
        saveList(menu, MENU_FILE);
        saveList(orders, ORDERS_FILE);
    }

    @SuppressWarnings("unchecked")
    private <T> List<T> loadList(String filename) {
        try (ObjectInputStream in = new ObjectInputStream(new FileInputStream(filename))) {
            return (List<T>) in.readObject();
        } catch (Exception e) {
            return new ArrayList<>();
        }
    }

    private <T> void saveList(List<T> list, String filename) {
        try (ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream(filename))) {
            out.writeObject(list);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public static void main(String[] args) {
        SwingUtilities.invokeLater(() -> {
            CafeManagementSystem app = new CafeManagementSystem();
            app.setVisible(true);
        });
    }
}
