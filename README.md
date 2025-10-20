# e-converse-sql
# CÓDIGO COMPLETO DE SQL PARA ECOMERCE

-- ======================================================
-- Schema: ecommerce (PostgreSQL)
-- Descrição: modelo lógico para cenário de e-commerce com:
--   - Cliente/Conta PF ou PJ (mutuamente exclusivo)
--   - Vários meios de pagamento por conta
--   - Entrega com status e código de rastreio
--   - Relacionamentos: produtos, fornecedores, vendedores, pedidos, itens, estoque
-- ======================================================

-- ------------------------------------------------------
-- 1. Cria schema
-- ------------------------------------------------------
CREATE SCHEMA IF NOT EXISTS ecommerce;
SET search_path = ecommerce, public;

-- ------------------------------------------------------
-- 2. Tabelas base
-- ------------------------------------------------------

-- accounts: representa conta do cliente (PF ou PJ) - mutual exclusivity via CHECK
CREATE TABLE accounts (
    account_id SERIAL PRIMARY KEY,
    account_type CHAR(2) NOT NULL CHECK (account_type IN ('PF','PJ')),
    email VARCHAR(255) NOT NULL UNIQUE,
    phone VARCHAR(30),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT now(),
    -- Campos específicos (aramazenados na mesma tabela para facilitar constraint de exclusividade)
    cpf VARCHAR(14),          -- para PF (formato: 000.000.000-00)
    full_name VARCHAR(255),   -- nome pessoa física
    cnpj VARCHAR(20),         -- para PJ (formato: 00.000.000/0000-00)
    company_name VARCHAR(255),-- nome empresa
    -- constraint para garantir que uma conta é PF ou PJ e que campos correspondem
    CHECK (
        (account_type = 'PF' AND cpf IS NOT NULL AND full_name IS NOT NULL AND cnpj IS NULL AND company_name IS NULL)
        OR
        (account_type = 'PJ' AND cnpj IS NOT NULL AND company_name IS NOT NULL AND cpf IS NULL AND full_name IS NULL)
    )
);

-- addresses: endereços (um cliente pode ter vários)
CREATE TABLE addresses (
    address_id SERIAL PRIMARY KEY,
    account_id INT NOT NULL REFERENCES accounts(account_id) ON DELETE CASCADE,
    label VARCHAR(50), -- ex: "Residencial", "Comercial"
    street VARCHAR(255),
    number VARCHAR(50),
    complement VARCHAR(255),
    district VARCHAR(100),
    city VARCHAR(100),
    state VARCHAR(100),
    zipcode VARCHAR(20),
    country VARCHAR(100) DEFAULT 'Brasil'
);

-- suppliers: fornecedores
CREATE TABLE suppliers (
    supplier_id SERIAL PRIMARY KEY,
    supplier_name VARCHAR(255) NOT NULL,
    contact_email VARCHAR(255),
    contact_phone VARCHAR(30),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT now()
);

-- sellers: vendedores (pessoa/empresa que vende no marketplace)
CREATE TABLE sellers (
    seller_id SERIAL PRIMARY KEY,
    seller_name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE,
    phone VARCHAR(30)
);

-- products: produtos (cada produto tem um fornecedor principal)
CREATE TABLE products (
    product_id SERIAL PRIMARY KEY,
    sku VARCHAR(50) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    price NUMERIC(12,2) NOT NULL CHECK (price >= 0),
    supplier_id INT REFERENCES suppliers(supplier_id) ON DELETE SET NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT now()
);

-- product_suppliers (many-to-many caso produto tenha múltiplos fornecedores)
CREATE TABLE product_suppliers (
    product_id INT NOT NULL REFERENCES products(product_id) ON DELETE CASCADE,
    supplier_id INT NOT NULL REFERENCES suppliers(supplier_id) ON DELETE CASCADE,
    supplier_sku VARCHAR(100),
    lead_time_days INT DEFAULT 0,
    PRIMARY KEY (product_id, supplier_id)
);

-- stock: estoque por produto (pode existir estoque por warehouse, mas simplificamos)
CREATE TABLE stock (
    stock_id SERIAL PRIMARY KEY,
    product_id INT NOT NULL REFERENCES products(product_id) ON DELETE CASCADE,
    quantity INT NOT NULL DEFAULT 0 CHECK (quantity >= 0),
    last_updated TIMESTAMP WITH TIME ZONE DEFAULT now(),
    UNIQUE (product_id)
);

-- payment_methods: várias formas de pagamento por account
CREATE TABLE payment_methods (
    payment_method_id SERIAL PRIMARY KEY,
    account_id INT NOT NULL REFERENCES accounts(account_id) ON DELETE CASCADE,
    method_type VARCHAR(50) NOT NULL, -- ex: 'Cartão', 'Boleto', 'PayPal'
    provider VARCHAR(100), -- ex: 'Visa', 'Mastercard', 'Banco X'
    details JSONB, -- dados como último4, validade, etc. (sensível: em produção criptografar)
    is_default BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT now()
);

-- orders: pedidos
CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    account_id INT NOT NULL REFERENCES accounts(account_id) ON DELETE RESTRICT,
    seller_id INT REFERENCES sellers(seller_id), -- opcional: pedido feito a um vendedor
    order_date TIMESTAMP WITH TIME ZONE DEFAULT now(),
    status VARCHAR(50) NOT NULL DEFAULT 'Pendente', -- ex: Pendente, Confirmado, Enviado, Entregue, Cancelado
    payment_method_id INT REFERENCES payment_methods(payment_method_id),
    -- delivery info (separado em order_delivery para normalização, mas mantemos FK aqui)
    total_amount NUMERIC(14,2) DEFAULT 0 CHECK (total_amount >= 0)
);

-- order_delivery: entrega com status e código rastreio
CREATE TABLE order_delivery (
    delivery_id SERIAL PRIMARY KEY,
    order_id INT UNIQUE NOT NULL REFERENCES orders(order_id) ON DELETE CASCADE,
    carrier VARCHAR(100), -- transportadora
    tracking_code VARCHAR(100),
    delivery_status VARCHAR(50) DEFAULT 'Aguardando', -- ex: Aguardando, Em Rota, Entregue, Extraviado
    estimated_delivery_date DATE,
    shipped_at TIMESTAMP WITH TIME ZONE
);

-- order_items: itens do pedido
CREATE TABLE order_items (
    order_item_id SERIAL PRIMARY KEY,
    order_id INT NOT NULL REFERENCES orders(order_id) ON DELETE CASCADE,
    product_id INT NOT NULL REFERENCES products(product_id),
    unit_price NUMERIC(12,2) NOT NULL CHECK (unit_price >= 0),
    quantity INT NOT NULL CHECK (quantity > 0),
    discount NUMERIC(12,2) DEFAULT 0 CHECK (discount >= 0),
    subtotal NUMERIC(14,2) GENERATED ALWAYS AS ( (unit_price * quantity) - discount ) STORED
);

-- supplier_sellers: tabela assistente para saber se algum vendedor também é fornecedor
CREATE TABLE supplier_sellers (
    supplier_id INT NOT NULL REFERENCES suppliers(supplier_id) ON DELETE CASCADE,
    seller_id INT NOT NULL REFERENCES sellers(seller_id) ON DELETE CASCADE,
    PRIMARY KEY (supplier_id, seller_id)
);

-- payments: registros de pagamento (para histórico / conciliação)
CREATE TABLE payments (
    payment_id SERIAL PRIMARY KEY,
    order_id INT NOT NULL REFERENCES orders(order_id) ON DELETE CASCADE,
    payment_date TIMESTAMP WITH TIME ZONE DEFAULT now(),
    amount NUMERIC(14,2) NOT NULL CHECK (amount >= 0),
    status VARCHAR(50) DEFAULT 'Pendente' -- Pendente, Aprovado, Recusado, Estornado
);

-- ------------------------------------------------------
-- 3. Índices úteis
-- ------------------------------------------------------
CREATE INDEX idx_products_supplier ON products(supplier_id);
CREATE INDEX idx_stock_product ON stock(product_id);
CREATE INDEX idx_orders_account ON orders(account_id);
CREATE INDEX idx_order_items_order ON order_items(order_id);

-- ------------------------------------------------------
-- 4. Inserts de exemplo (dados de teste)
-- ------------------------------------------------------

-- Accounts (PF e PJ)
INSERT INTO accounts (account_type, email, phone, cpf, full_name)
VALUES
('PF','ana@example.com','+55 11 99999-0001','123.456.789-00','Ana Silva'),
('PF','joao@example.com','+55 21 98888-1111','987.654.321-00','João Pereira'),
('PJ','lojax@example.com','+55 11 3333-4444',NULL,NULL,'12.345.678/0001-99','Loja X Ltda');

-- Addresses
INSERT INTO addresses (account_id, label, street, number, district, city, state, zipcode)
VALUES
(1,'Residencial','Rua A','100','Centro','São Paulo','SP','01000-000'),
(2,'Comercial','Av. B','200','Centro','Rio de Janeiro','RJ','20000-000'),
(3,'Comercial','Av. Loja','1','Bairro','São Paulo','SP','01100-000');

-- Suppliers
INSERT INTO suppliers (supplier_name, contact_email, contact_phone)
VALUES
('Fornecedora Alfa','contato@alfa.com.br','+55 11 4002-0001'),
('Fornecedor Beta','contato@beta.com.br','+55 21 4002-0002');

-- Sellers
INSERT INTO sellers (seller_name, email, phone)
VALUES
('Vendedor 1','v1@market.com','+55 11 7700-0001'),
('Fornecedor Beta','beta@seller.com','+55 21 4002-0002'); -- note: seller name similar to supplier for demo

-- Products
INSERT INTO products (sku, name, description, price, supplier_id)
VALUES
('SKU-100','Teclado Mecânico','Teclado mecânico RGB',250.00,1),
('SKU-200','Mouse Gamer','Mouse com sensor óptico',150.00,2),
('SKU-300','Monitor 24"','Monitor FHD 24 polegadas',900.00,1);

-- product_suppliers (m-to-m)
INSERT INTO product_suppliers (product_id, supplier_id, supplier_sku, lead_time_days)
VALUES
(1,1,'ALFA-T100',5),
(2,2,'BETA-M200',7),
(3,1,'ALFA-M300',10);

-- Stock
INSERT INTO stock (product_id, quantity) VALUES
(1, 50),
(2, 30),
(3, 15);

-- supplier_sellers (indicar que seller 2 também é fornecedor 2)
-- vamos relacionar supplier_id 2 com seller_id 2
INSERT INTO supplier_sellers (supplier_id, seller_id) VALUES (2, 2);

-- Payment methods
INSERT INTO payment_methods (account_id, method_type, provider, details, is_default)
VALUES
(1,'Cartão','Visa','{"last4":"1111","exp":"12/26"}',true),
(1,'PayPal','PayPal','{}',false),
(3,'Boleto','Banco X','{}',true);

-- Orders
INSERT INTO orders (account_id, seller_id, order_date, status, payment_method_id)
VALUES
(1,1,'2025-10-01 10:00:00+00','Confirmado',1),
(2,NULL,'2025-10-03 15:30:00+00','Pendente',NULL),
(3,2,'2025-10-05 09:20:00+00','Enviado',3);

-- Order items (note: subtotal generated)
INSERT INTO order_items (order_id, product_id, unit_price, quantity, discount)
VALUES
(1,1,250.00,1,0),
(1,2,150.00,2,10.00),
(2,2,150.00,1,0),
(3,3,900.00,1,50.00);

-- Atualizar total_amount dos pedidos com base nos itens
UPDATE orders o
SET total_amount = COALESCE(sub.total,0)
FROM (
    SELECT order_id, SUM((unit_price * quantity) - discount) AS total
    FROM order_items
    GROUP BY order_id
) sub
WHERE o.order_id = sub.order_id;

-- Order delivery
INSERT INTO order_delivery (order_id, carrier, tracking_code, delivery_status, estimated_delivery_date, shipped_at)
VALUES
(1,'Transportadora X','TRK12345','Entregue','2025-10-07','2025-10-06 08:00:00+00'),
(3,'Transportadora Y','TRK98765','Em Rota','2025-10-12','2025-10-06 10:00:00+00');

-- Payments
INSERT INTO payments (order_id, payment_date, amount, status)
VALUES
(1,'2025-10-01 10:05:00+00',540.00,'Aprovado'), -- 250 + (150*2 -10) = 540
(3,'2025-10-05 09:25:00+00',850.00,'Pendente'); -- 900 - 50 = 850

-- ------------------------------------------------------
-- 5. Queries demonstrativas (responder às questões e mostrar uso de WHERE, HAVING, JOIN, ORDER BY, atributos derivados)
-- ------------------------------------------------------

-- Q1: Quantos pedidos foram feitos por cada cliente?
-- Pergunta: "Quantos pedidos cada conta realizou?"
SELECT a.account_id, 
       COALESCE(a.full_name, a.company_name) AS account_name,
       COUNT(o.order_id) AS total_pedidos,
       SUM(o.total_amount) AS total_gasto
FROM accounts a
LEFT JOIN orders o ON o.account_id = a.account_id
GROUP BY a.account_id, account_name
ORDER BY total_pedidos DESC;

-- Q2: Algum vendedor também é fornecedor?
-- Pergunta: "Quais sellers também aparecem como suppliers (via supplier_sellers ou nomes semelhantes)?"
-- Abordagem 1: via tabela de vínculo
SELECT s.seller_id, s.seller_name, ss.supplier_id, sup.supplier_name
FROM sellers s
JOIN supplier_sellers ss ON ss.seller_id = s.seller_id
JOIN suppliers sup ON sup.supplier_id = ss.supplier_id;

-- Abordagem 2: tentativa por nome parecido (ex.: string match)
SELECT s.seller_id, s.seller_name, sup.supplier_id, sup.supplier_name
FROM sellers s
JOIN suppliers sup ON lower(s.seller_name) = lower(sup.supplier_name);

-- Q3: Relação de produtos, fornecedores e estoques
-- Pergunta: "Lista de produtos com nome do fornecedor principal e quantidade em estoque"
SELECT p.product_id, p.sku, p.name AS produto, 
       sup.supplier_name AS fornecedor_principal,
       COALESCE(st.quantity,0) AS quantidade_estoque,
       p.price
FROM products p
LEFT JOIN suppliers sup ON sup.supplier_id = p.supplier_id
LEFT JOIN stock st ON st.product_id = p.product_id
ORDER BY p.name;

-- Q4: Relação de nomes dos fornecedores e nomes dos produtos (muitos-para-muitos)
-- Pergunta: "Todos os fornecedores e quais produtos eles fornecem"
SELECT sup.supplier_id, sup.supplier_name, p.product_id, p.name AS produto
FROM suppliers sup
JOIN product_suppliers ps ON ps.supplier_id = sup.supplier_id
JOIN products p ON p.product_id = ps.product_id
ORDER BY sup.supplier_name, p.name;

-- Q5: Pedidos com valor total (atributo derivado) e quantidade de itens
-- Pergunta: "Mostrar pedidos com total calculado e número de itens, filtrar por total > 500"
SELECT o.order_id,
       o.order_date,
       COALESCE(SUM(oi.unit_price * oi.quantity - oi.discount),0) AS total_calculado,
       COUNT(oi.order_item_id) AS itens_qtd
FROM orders o
LEFT JOIN order_items oi ON oi.order_id = o.order_id
GROUP BY o.order_id, o.order_date
HAVING COALESCE(SUM(oi.unit_price * oi.quantity - oi.discount),0) > 500
ORDER BY total_calculado DESC;

-- Q6: Clientes que mais gastaram (top 5)
SELECT a.account_id, COALESCE(a.full_name, a.company_name) AS nome, SUM(o.total_amount) AS gasto_total
FROM accounts a
JOIN orders o ON o.account_id = a.account_id
GROUP BY a.account_id, nome
ORDER BY gasto_total DESC
LIMIT 5;

-- Q7: Produtos com estoque abaixo de um limite (WHERE on derived/aggregated join)
SELECT p.product_id, p.name, COALESCE(st.quantity,0) AS estoque
FROM products p
LEFT JOIN stock st ON st.product_id = p.product_id
WHERE COALESCE(st.quantity,0) < 20
ORDER BY estoque ASC;

-- Q8: Pedidos e status de entrega (JOIN com order_delivery) - mostra traço de rastreio
SELECT o.order_id, COALESCE(a.full_name,a.company_name) AS cliente, o.status AS pedido_status,
       od.delivery_status, od.tracking_code
FROM orders o
LEFT JOIN accounts a ON a.account_id = o.account_id
LEFT JOIN order_delivery od ON od.order_id = o.order_id
ORDER BY o.order_date DESC;

-- Q9: Vendas por fornecedor (soma dos itens vendidos de produtos do fornecedor)
SELECT sup.supplier_id, sup.supplier_name, 
       SUM(oi.unit_price * oi.quantity - oi.discount) AS receita_por_fornecedor,
       COUNT(DISTINCT oi.order_id) AS pedidos_relacionados
FROM suppliers sup
JOIN products p ON p.supplier_id = sup.supplier_id
JOIN order_items oi ON oi.product_id = p.product_id
JOIN orders o ON o.order_id = oi.order_id
GROUP BY sup.supplier_id, sup.supplier_name
ORDER BY receita_por_fornecedor DESC;

-- Q10: Média de items por pedido (atributo agregado) e filtrar com HAVING (ex.: médias maiores que 1)
SELECT o.order_id,
       COUNT(oi.order_item_id) AS itens_qtd,
       AVG(oi.quantity) AS media_qtd_por_item
FROM orders o
LEFT JOIN order_items oi ON oi.order_id = o.order_id
GROUP BY o.order_id
HAVING COUNT(oi.order_item_id) > 0
ORDER BY itens_qtd DESC;

-- Q11: Exemplo de subquery correlacionada: Último pedido de cada cliente
SELECT a.account_id, COALESCE(a.full_name,a.company_name) AS cliente,
       (SELECT o2.order_id FROM orders o2 WHERE o2.account_id = a.account_id ORDER BY o2.order_date DESC LIMIT 1) AS ultimo_pedido_id
FROM accounts a;

-- Q12: Query que responde "Quantos pedidos foram feitos por cada cliente PF (somente PF), ordenado"
SELECT a.account_id, a.full_name AS nome, COUNT(o.order_id) AS pedidos
FROM accounts a
JOIN orders o ON o.account_id = a.account_id
WHERE a.account_type = 'PF'
GROUP BY a.account_id, a.full_name
ORDER BY pedidos DESC;

-- Q13: Verificar integridade: pedidos sem método de pagamento (WHERE)
SELECT o.order_id, o.order_date, o.total_amount, a.email
FROM orders o
JOIN accounts a ON a.account_id = o.account_id
WHERE o.payment_method_id IS NULL;

-- Q14: Query combinada com expressions: aplicar imposto de 18% e frete fixo de R$20 como atributos derivados
SELECT o.order_id, o.total_amount,
       o.total_amount * 0.18 AS imposto_18pct,
       o.total_amount + (o.total_amount * 0.18) + 20 AS total_com_imposto_frete
FROM orders o
ORDER BY total_com_imposto_frete DESC;

-- ------------------------------------------------------
-- 6. Observações / recomendações (em texto) para o README
-- ------------------------------------------------------
/*
README SUGGESTION (adicione ao seu repo GitHub):

# Projeto Lógico — E-commerce (Modelo e Scripts SQL)

## Descrição
Este repositório contém a modelagem lógica de um sistema de e-commerce, contemplando:
- Contas de clientes (PF e PJ) — garantimos exclusividade de tipo com constraints;
- Pagamentos: múltiplos meios por conta;
- Entregas com status e código de rastreio;
- Produtos, fornecedores, vendedores, estoque, pedidos, itens e histórico de pagamentos.

## Arquivos
- schema_ecommerce.sql  -> Script completo de criação (DDL), inserts de teste (DML) e queries de exemplo.
- README.md             -> Este arquivo com instruções e explicações.

## Observações rápidas sobre o modelo
- A tabela `accounts` contém `account_type` ('PF' ou 'PJ') e campos específicos armazenados com `CHECK` para garantir exclusividade.
- `payment_methods` permite múltiplas formas de pagamento por conta.
- `order_delivery` contém `delivery_status` e `tracking_code`.
- Relações N:N: `product_suppliers` (produto <-> fornecedor), `supplier_sellers` (fornecedor <-> seller).
- Em produção, campos sensíveis (ex: dados de cartão) devem ser criptografados e regulados (LGPD/PCI-DSS).

## Como rodar
1. Criar database no PostgreSQL: `CREATE DATABASE ecommerce_db;`
2. Conectar e executar `schema_ecommerce.sql`.
3. Rodar as queries de exemplo para validação.

## Possíveis extensões
- Separar PF/PJ em tabelas especializadas com triggers para garantir exclusividade.
- Adicionar tabelas de warehouse, logs de estoque, status history para pedidos/entregas.
- Implementar views para relatórios (vendas por dia, top produtos, etc).

*/

-- FIM DO SCRIPT


