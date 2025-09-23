import { useState } from "react";
import { Button } from "@/components/ui/button";
import { Card, CardContent } from "@/components/ui/card";
import { motion } from "framer-motion";

export default function LandingPage() {
  const [formData, setFormData] = useState({
    name: "",
    email: "",
    phone: "",
    insuranceType: "",
  });

  const insurances = [
    { name: "Final Expense Insurance", desc: "Protect your loved ones with peace of mind." },
    { name: "Medicare Plans", desc: "Comprehensive healthcare coverage for seniors." },
    { name: "Auto Insurance", desc: "Drive with confidence knowing you’re covered." },
    { name: "Affordable Care Act (ACA)", desc: "Affordable health insurance options for all." },
    { name: "Indexed Universal Life (IUL)", desc: "Secure your future with life insurance + growth." },
    { name: "Home Insurance", desc: "Protect your home, family, and valuables." },
  ];

  const handleSubmit = (e) => {
    e.preventDefault();
    // Journaya Integration Placeholder
    console.log("Form submitted", formData);
    alert("Form submitted with Journaya integration!");
  };

  return (
    <div className="min-h-screen bg-gradient-to-b from-blue-50 to-blue-100 text-gray-900">
      {/* Hero Section */}
      <section className="text-center py-20 px-6 bg-blue-700 text-white">
        <motion.h1
          initial={{ opacity: 0, y: -30 }}
          animate={{ opacity: 1, y: 0 }}
          className="text-4xl md:text-6xl font-bold"
        >
          Get Yourself Insured
        </motion.h1>
        <p className="mt-4 text-lg md:text-2xl">
          Affordable. Reliable. Tailored Insurance Plans for You.
        </p>
        <Button className="mt-6 px-6 py-3 bg-yellow-400 text-black rounded-xl shadow-lg hover:bg-yellow-300">
          Get Your Free Quote
        </Button>
      </section>

      {/* Insurance Options */}
      <section className="py-16 px-6 max-w-6xl mx-auto grid gap-8 md:grid-cols-3">
        {insurances.map((item, index) => (
          <motion.div
            key={index}
            whileHover={{ scale: 1.05 }}
            className="bg-white rounded-2xl shadow-md p-6"
          >
            <h2 className="text-xl font-semibold">{item.name}</h2>
            <p className="mt-2 text-gray-600">{item.desc}</p>
          </motion.div>
        ))}
      </section>

      {/* Lead Form (Jornaya Integration Placeholder) */}
      <section className="bg-white py-16 px-6">
        <h2 className="text-3xl font-bold text-center mb-6">
          Get Your Free Insurance Quote
        </h2>
        <form
          onSubmit={handleSubmit}
          className="max-w-lg mx-auto space-y-4 bg-gray-50 p-6 rounded-2xl shadow-md"
        >
          <input
            type="text"
            placeholder="Full Name"
            className="w-full p-3 border rounded-xl"
            value={formData.name}
            onChange={(e) => setFormData({ ...formData, name: e.target.value })}
            required
          />
          <input
            type="email"
            placeholder="Email Address"
            className="w-full p-3 border rounded-xl"
            value={formData.email}
            onChange={(e) => setFormData({ ...formData, email: e.target.value })}
            required
          />
          <input
            type="tel"
            placeholder="Phone Number"
            className="w-full p-3 border rounded-xl"
            value={formData.phone}
            onChange={(e) => setFormData({ ...formData, phone: e.target.value })}
            required
          />
          <select
            className="w-full p-3 border rounded-xl"
            value={formData.insuranceType}
            onChange={(e) =>
              setFormData({ ...formData, insuranceType: e.target.value })
            }
            required
          >
            <option value="">Select Insurance Type</option>
            {insurances.map((item, idx) => (
              <option key={idx} value={item.name}>
                {item.name}
              </option>
            ))}
          </select>
          {/* Journaya Token Placeholder */}
          <input type="hidden" name="jornaya_leadid" value="{LEAD_ID}" />
          <Button
            type="submit"
            className="w-full py-3 bg-blue-600 text-white rounded-xl hover:bg-blue-500"
          >
            Submit Quote Request
          </Button>
        </form>
      </section>

      {/* Why Choose Us */}
      <section className="py-16 px-6 max-w-5xl mx-auto text-center">
        <h2 className="text-3xl font-bold">Why Choose Get Yourself Insured?</h2>
        <div className="grid gap-8 mt-8 md:grid-cols-3">
          <Card>
            <CardContent className="p-6">
              <h3 className="font-semibold text-xl">Trusted Partners</h3>
              <p className="text-gray-600 mt-2">
                We work with leading insurance providers nationwide.
              </p>
            </CardContent>
          </Card>
          <Card>
            <CardContent className="p-6">
              <h3 className="font-semibold text-xl">Affordable Coverage</h3>
              <p className="text-gray-600 mt-2">
                Plans designed to fit your needs and budget.
              </p>
            </CardContent>
          </Card>
          <Card>
            <CardContent className="p-6">
              <h3 className="font-semibold text-xl">Fast & Easy</h3>
              <p className="text-gray-600 mt-2">
                Get your quotes in minutes with minimal hassle.
              </p>
            </CardContent>
          </Card>
        </div>
      </section>

      {/* Footer */}
      <footer className="bg-blue-700 text-white py-6 text-center">
        <p>© {new Date().getFullYear()} Get Yourself Insured. All Rights Reserved.</p>
      </footer>
    </div>
  );
}
