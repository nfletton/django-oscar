======================
How to handle US taxes
======================

When trading in the US, taxes aren't known until the customer's shipping
address has been entered.  This scenario requires two changes from core Oscar.

Ensure your site strategy returns prices without taxes applied
--------------------------------------------------------------

First, the site strategy should return all prices without tax when the customer
is based in the US.  Oscar provides a :class:`~oscar.apps.partner.strategy.US`
strategy class that uses the :class:`~oscar.apps.partner.strategy.DeferredTax`
mixin to indicate that prices don't include taxes.

See :doc:`the documentation on strategies </topics/prices_and_availability>`
for further guidance on how to replace strategies.

Adjust checkout views to apply taxes once they are known
--------------------------------------------------------

Second, the :class:`~oscar.apps.checkout.session.CheckoutSessionMixin`
should be overridden within your project to apply taxes
to the submission.

.. code-block:: python

    from oscar.apps.checkout import session
    from apps import tax

    # Override the session mixin (which every checkout view uses) so we can apply
    # taxes when the shipping address is known.
    class CheckoutSessionMixin(session.CheckoutSessionMixin):

        def build_submission(self, **kwargs):
            submission = super(CheckoutSessionMixin, self).build_submission(
                **kwargs)

            if submission['shipping_address'] and submission['shipping_method']:
                tax.apply_to(submission)

                # Recalculate order total to ensure we have a tax-inclusive total
                submission['order_total'] = self.get_order_totals(
                    submission['basket'], submission['shipping_charge'])

            return submission

An example implementation of the ``tax.py`` module is:

.. code-block:: python

    from decimal import Decimal as D

    # State tax rates
    STATE_TAX_RATES = {
        'NJ': D('0.07')
    }
    ZERO = D('0.00')

    def apply_to(submission):
        """
        Calculate and apply taxes to a submission
        """
        # This is a dummy tax function which only applies taxes for addresses in
        # New Jersey and New York. In reality, you'll probably want to use a tax
        # service like Avalara to look up the taxes for a given submission.
        shipping_address = submission['shipping_address']
        rate = STATE_TAX_RATES.get(
            shipping_address.state, ZERO)
        for line in submission['basket'].all_lines():
            line_tax = calculate_tax(
                line.line_price_excl_tax_incl_discounts, rate)
            # We need to split the line tax down into a unit tax amount.
            unit_tax = (line_tax / line.quantity).quantize(D('0.01'))
            line.purchase_info.price.tax = unit_tax

        shipping_charge = submission['shipping_charge']
        if shipping_charge is not None:
            shipping_charge.tax = calculate_tax(
                shipping_charge.excl_tax, rate)


    def calculate_tax(price, rate):
        tax = price * rate
        return tax.quantize(D('0.01'))

.. tip::

   Oscar's repository contains a sample Oscar site customised for the US.  See
   :ref:`us_site` for more information.
