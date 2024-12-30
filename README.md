import 'package:flutter/material.dart';
import 'package:http/http.dart' as http;
import 'dart:convert';

void main() {
  runApp(CurrencyConverterApp());
}

class CurrencyConverterApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Currency Converter',
      theme: ThemeData(primarySwatch: Colors.blue),
      home: CurrencyConverter(),
    );
  }
}

class CurrencyConverter extends StatefulWidget {
  @override
  _CurrencyConverterState createState() => _CurrencyConverterState();
}

class _CurrencyConverterState extends State<CurrencyConverter> {
  final TextEditingController _amountController = TextEditingController();
  final List<String> _currencies = ['USD', 'EUR', 'GBP', 'INR', 'JPY', 'AUD'];
  String _sourceCurrency = 'USD';
  String _targetCurrency = 'INR';
  String? _conversionResult;
  String? _taxComment;
  String? _paymentComment;
  bool _isLoading = false;
  String? _errorMessage;
  double? _convertedValue;
  List<String> _conversionHistory = [];

  // Fetch and convert currency
  Future<void> _convertCurrency() async {
    setState(() {
      _isLoading = true;
      _errorMessage = null;
      _conversionResult = null;
    });

    final amount = double.tryParse(_amountController.text);
    if (amount == null || amount <= 0) {
      setState(() {
        _isLoading = false;
        _errorMessage = 'Please enter a valid amount.';
      });
      return;
    }

    final url = Uri.parse(
        'https://api.exchangerate-api.com/v4/latest/$_sourceCurrency');
    try {
      final response = await http.get(url);
      if (response.statusCode == 200) {
        final data = json.decode(response.body);
        final rates = data['rates'];
        final rate = rates[_targetCurrency];
        if (rate != null) {
          final result = amount * rate;

          setState(() {
            _convertedValue = result;
            _conversionResult =
                '$amount $_sourceCurrency = ${result.toStringAsFixed(2)} $_targetCurrency';
            _conversionHistory.add(_conversionResult!);
          });
        } else {
          setState(() {
            _errorMessage = 'Unable to fetch conversion rate.';
          });
        }
      } else {
        setState(() {
          _errorMessage = 'Failed to fetch data from the API.';
        });
      }
    } catch (e) {
      setState(() {
        _errorMessage = 'An error occurred. Please try again.';
      });
    } finally {
      setState(() {
        _isLoading = false;
      });
    }
  }

  // Calculate tax and amount to be paid
  void _calculateTaxAndPayment(double percentage) {
    if (_convertedValue != null) {
      final tax = (_convertedValue! * percentage) / 100;
      final amountToPay = _convertedValue! - tax;

      setState(() {
        _taxComment =
            'Total Tax: ${tax.toStringAsFixed(2)} $_targetCurrency (${percentage.toInt()}%)';
        _paymentComment =
            'Amount to be paid: ${amountToPay.toStringAsFixed(2)} $_targetCurrency';
      });
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('Currency Converter'),
      ),
      body: Padding(
        padding: const EdgeInsets.all(16.0),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text(
              'Enter Amount:',
              style: TextStyle(fontSize: 16, fontWeight: FontWeight.bold),
            ),
            TextField(
              controller: _amountController,
              keyboardType: TextInputType.number,
              decoration: InputDecoration(
                border: OutlineInputBorder(),
                labelText: 'Amount',
              ),
            ),
            Row(
              children: [
                Expanded(
                  child: Column(
                    children: [
                      Text('From:'),
                      DropdownButtonFormField(
                        value: _sourceCurrency,
                        items: _currencies.map((currency) {
                          return DropdownMenuItem(
                            value: currency,
                            child: Text(currency),
                          );
                        }).toList(),
                        onChanged: (value) {
                          setState(() {
                            _sourceCurrency = value!;
                          });
                        },
                      ),
                    ],
                  ),
                ),
                Expanded(
                  child: Column(
                    children: [
                      Text('To:'),
                      DropdownButtonFormField(
                        value: _targetCurrency,
                        items: _currencies.map((currency) {
                          return DropdownMenuItem(
                            value: currency,
                            child: Text(currency),
                          );
                        }).toList(),
                        onChanged: (value) {
                          setState(() {
                            _targetCurrency = value!;
                          });
                        },
                      ),
                    ],
                  ),
                ),
              ],
            ),
            ElevatedButton(
              onPressed: _convertCurrency,
              child: Text('Convert Currency'),
            ),
            if (_conversionResult != null) ...[
              Text(
                _conversionResult!,
                style: TextStyle(fontSize: 18, color: Colors.green),
              ),
              Row(
                children: [
                  ElevatedButton(
                    onPressed: () => _calculateTaxAndPayment(20),
                    child: Text('20% Tax'),
                  ),
                  SizedBox(width: 16),
                  ElevatedButton(
                    onPressed: () => _calculateTaxAndPayment(30),
                    child: Text('30% Tax'),
                  ),
                ],
              ),
              if (_taxComment != null)
                Text(
                  _taxComment!,
                  style: TextStyle(fontSize: 16, color: Colors.blue),
                ),
              if (_paymentComment != null)
                Text(
                  _paymentComment!,
                  style: TextStyle(fontSize: 16, color: Colors.purple),
                ),
            ],
            if (_errorMessage != null)
              Text(
                _errorMessage!,
                style: TextStyle(color: Colors.red),
              ),
          ],
        ),
      ),
    );
  }
}
