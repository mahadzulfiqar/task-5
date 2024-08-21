# task-5
#include <iostream>
#include <fstream>
#include <curl/curl.h>
#include <gumbo.h>
#include <vector>
#include <string>

// Structure to store product information
struct Product {
    std::string name;
    std::string price;
    std::string rating;
};

// Function to write the curl data into a string
size_t WriteCallback(void* contents, size_t size, size_t nmemb, std::string* s) {
    size_t totalSize = size * nmemb;
    s->append((char*)contents, totalSize);
    return totalSize;
}

// Function to fetch the HTML content using libcurl
std::string fetchHTML(const std::string& url) {
    CURL* curl;
    CURLcode res;
    std::string readBuffer;

    curl = curl_easy_init();
    if (curl) {
        curl_easy_setopt(curl, CURLOPT_URL, url.c_str());
        curl_easy_setopt(curl, CURLOPT_WRITEFUNCTION, WriteCallback);
        curl_easy_setopt(curl, CURLOPT_WRITEDATA, &readBuffer);
        res = curl_easy_perform(curl);
        curl_easy_cleanup(curl);
    }

    return readBuffer;
}

// Function to search through the HTML and extract product information
void searchForProducts(GumboNode* node, std::vector<Product>& products) {
    if (node->type != GUMBO_NODE_ELEMENT) return;

    // Check if the current node is a product entry
    if (node->v.element.tag == GUMBO_TAG_DIV) {
        GumboAttribute* classAttr = gumbo_get_attribute(&node->v.element.attributes, "class");
        if (classAttr && std::string(classAttr->value).find("product") != std::string::npos) {
            Product product;
            // Example: Extracting product name, price, and rating
            for (size_t i = 0; i < node->v.element.children.length; ++i) {
                GumboNode* child = static_cast<GumboNode*>(node->v.element.children.data[i]);

                if (child->type == GUMBO_NODE_ELEMENT && child->v.element.tag == GUMBO_TAG_H2) {
                    GumboNode* nameNode = static_cast<GumboNode*>(child->v.element.children.data[0]);
                    product.name = nameNode->v.text.text;
                }

                if (child->type == GUMBO_NODE_ELEMENT && child->v.element.tag == GUMBO_TAG_SPAN) {
                    GumboAttribute* classAttr = gumbo_get_attribute(&child->v.element.attributes, "class");
                    if (classAttr && std::string(classAttr->value).find("price") != std::string::npos) {
                        GumboNode* priceNode = static_cast<GumboNode*>(child->v.element.children.data[0]);
                        product.price = priceNode->v.text.text;
                    }

                    if (classAttr && std::string(classAttr->value).find("rating") != std::string::npos) {
                        GumboNode* ratingNode = static_cast<GumboNode*>(child->v.element.children.data[0]);
                        product.rating = ratingNode->v.text.text;
                    }
                }
            }
            products.push_back(product);
        }
    }

    // Recurse through the children of the current node
    for (size_t i = 0; i < node->v.element.children.length; ++i) {
        searchForProducts(static_cast<GumboNode*>(node->v.element.children.data[i]), products);
    }
}

// Function to parse HTML and extract product information
std::vector<Product> parseHTML(const std::string& html) {
    std::vector<Product> products;
    GumboOutput* output = gumbo_parse(html.c_str());
    searchForProducts(output->root, products);
    gumbo_destroy_output(&kGumboDefaultOptions, output);
    return products;
}

// Function to save products to a CSV file
void saveToCSV(const std::vector<Product>& products, const std::string& filename) {
    std::ofstream file(filename);
    file << "Product Name,Price,Rating\n";
    for (const auto& product : products) {
        file << "\"" << product.name << "\",\"" << product.price << "\",\"" << product.rating << "\"\n";
    }
    file.close();
    std::cout << "Data saved to " << filename << std::endl;
}

int main() {
    std::string url = "https://example.com/products"; // Replace with the actual URL
    std::string htmlContent = fetchHTML(url);

    if (!htmlContent.empty()) {
        std::vector<Product> products = parseHTML(htmlContent);
        saveToCSV(products, "products.csv");
    } else {
        std::cerr << "Failed to fetch HTML content." << std::endl;
    }

    return 0;
}
