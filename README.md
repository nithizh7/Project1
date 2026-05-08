import { useState, useEffect } from 'react'
import { getAllDeliveries, getDeliveriesByAgent, completeDelivery } from '../api/deliveryApi'
import Table, { EmptyRow, TR, TD } from '../components/Table'
import StatusBadge from '../components/StatusBadge'
import LoadingSpinner from '../components/LoadingSpinner'
import { useAuth } from '../context/AuthContext'

export default function DeliveriesPage() {
  const { user }    = useAuth()
  const isDelivery  = user?.role === 'DELIVERY'

  const [deliveries, setDeliveries] = useState([])
  const [loading, setLoading]       = useState(true)

  const load = () => {
    setLoading(true)
    const fn = isDelivery ? () => getDeliveriesByAgent(user.id) : getAllDeliveries
    fn().then(r => setDeliveries(r.data)).finally(() => setLoading(false))
  }
  useEffect(load, [])

  const handleComplete = async id => {
    if (!confirm('Mark this delivery as completed?')) return
    try { await completeDelivery(id); load() }
    catch (e) { alert(e.response?.data?.message || 'Failed') }
  }

  return (
    <div>
      <h1 className="text-2xl font-bold text-gray-900 mb-6">{isDelivery ? 'My Deliveries' : 'Deliveries'}</h1>
      <div className="bg-white rounded-xl shadow-sm border border-gray-100">
        {loading ? <LoadingSpinner /> : (
          <Table headers={['ID', 'Farmer', 'Warehouse', 'Agent', 'Status', 'Delivered At', 'Action']}>
            {!deliveries.length ? <EmptyRow cols={7} /> : deliveries.map(d => (
              <TR key={d.id}>
                <TD mono>#{d.id}</TD>
                <TD><span className="font-medium text-gray-900">{d.farmerName}</span></TD>
                <TD>{d.warehouseName}</TD>
                <TD>{d.deliveryAgentName}</TD>
                <TD><StatusBadge status={d.status} /></TD>
                <TD>{d.deliveredAt ? new Date(d.deliveredAt).toLocaleString() : '—'}</TD>
                <td className="px-4 py-3">
                  {d.status === 'IN_PROGRESS' && (
                    <button onClick={() => handleComplete(d.id)}
                      className="bg-green-600 hover:bg-green-700 text-white rounded-lg px-3 py-1.5 text-xs font-medium">
                      ✓ Mark Complete
                    </button>
                  )}
                </td>
              </TR>
            ))}
          </Table>
        )}
      </div>
    </div>
  )
}















import { useState, useEffect } from 'react'
import { getAllInvoices, getInvoicesByFarmer, payInvoice } from '../api/invoiceApi'
import Table, { EmptyRow, TR, TD } from '../components/Table'
import StatusBadge from '../components/StatusBadge'
import Modal from '../components/Modal'
import LoadingSpinner from '../components/LoadingSpinner'
import { useAuth } from '../context/AuthContext'

const PAYMENT_METHODS = ['CASH', 'BANK_TRANSFER', 'MOBILE_MONEY']

export default function InvoicesPage() {
  const { user }  = useAuth()
  const isFarmer  = user?.role === 'FARMER'

  const [invoices, setInvoices] = useState([])
  const [loading, setLoading]   = useState(true)
  const [showPay, setShowPay]   = useState(false)
  const [selected, setSelected] = useState(null)
  const [method, setMethod]     = useState('CASH')
  const [paying, setPaying]     = useState(false)
  const [error, setError]       = useState('')

  const load = () => {
    setLoading(true)
    const fn = isFarmer ? () => getInvoicesByFarmer(user.id) : getAllInvoices
    fn().then(r => setInvoices(r.data)).finally(() => setLoading(false))
  }
  useEffect(load, [])

  const handlePay = async e => {
    e.preventDefault(); setPaying(true); setError('')
    try {
      await payInvoice(selected.id, { paymentMethod: method })
      setShowPay(false); load()
    } catch (err) { setError(err.response?.data?.message || 'Payment failed') }
    finally { setPaying(false) }
  }

  return (
    <div>
      <h1 className="text-2xl font-bold text-gray-900 mb-6">{isFarmer ? 'My Invoices' : 'Invoices'}</h1>
      <div className="bg-white rounded-xl shadow-sm border border-gray-100">
        {loading ? <LoadingSpinner /> : (
          <Table headers={['ID', 'Farmer', 'Request', 'Amount', 'Status', 'Issued', 'Paid', 'Action']}>
            {!invoices.length ? <EmptyRow cols={8} /> : invoices.map(inv => (
              <TR key={inv.id}>
                <TD mono>#{inv.id}</TD>
                <TD><span className="font-medium text-gray-900">{inv.farmerName}</span></TD>
                <TD mono>#{inv.requestId}</TD>
                <TD><span className="font-bold text-gray-900">₹{Number(inv.totalAmount).toLocaleString()}</span></TD>
                <TD><StatusBadge status={inv.status} /></TD>
                <TD>{inv.issuedAt ? new Date(inv.issuedAt).toLocaleDateString() : '—'}</TD>
                <TD>{inv.paidAt ? new Date(inv.paidAt).toLocaleDateString() : '—'}</TD>
                <td className="px-4 py-3">
                  {inv.status === 'UNPAID' && (
                    <button onClick={() => { setSelected(inv); setMethod('CASH'); setError(''); setShowPay(true) }}
                      className="bg-blue-700 hover:bg-blue-800 text-white rounded-lg px-3 py-1.5 text-xs font-medium">
                      💳 Pay
                    </button>
                  )}
                </td>
              </TR>
            ))}
          </Table>
        )}
      </div>

      {showPay && selected && (
        <Modal title={`Pay Invoice #${selected.id}`} onClose={() => setShowPay(false)}>
          <div className="bg-gray-50 rounded-lg p-4 text-center mb-5">
            <p className="text-sm text-gray-500 mb-1">Amount Due</p>
            <p className="text-3xl font-bold text-gray-900">₹{Number(selected.totalAmount).toLocaleString()}</p>
          </div>
          {error && <div className="mb-3 px-3 py-2 bg-red-50 text-red-700 text-sm rounded-lg">{error}</div>}
          <form onSubmit={handlePay} className="space-y-4">
            <div>
              <label className="block text-sm text-gray-600 mb-1">Payment Method *</label>
              <select className="w-full border border-gray-300 rounded-lg px-3 py-2 text-sm focus:outline-none focus:ring-2 focus:ring-blue-500"
                value={method} onChange={e => setMethod(e.target.value)} required>
                {PAYMENT_METHODS.map(m => <option key={m} value={m}>{m.replace('_', ' ')}</option>)}
              </select>
            </div>
            <div className="flex justify-end gap-2">
              <button type="button" onClick={() => setShowPay(false)}
                className="border border-gray-300 hover:bg-gray-50 rounded-lg px-4 py-2 text-sm">Cancel</button>
              <button type="submit" disabled={paying}
                className="bg-blue-700 hover:bg-blue-800 text-white rounded-lg px-4 py-2 text-sm font-medium disabled:opacity-60">
                {paying ? 'Processing…' : 'Confirm Payment'}
              </button>
            </div>
          </form>
        </Modal>
      )}
    </div>
  )
}
