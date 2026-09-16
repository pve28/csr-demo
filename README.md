import { useState, useEffect } from 'react'
import { createClient } from '@supabase/supabase-js'

const supabase = createClient(process.env.NEXT_PUBLIC_SUPABASE_URL, process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY)

export default function Home(){
  const [role,setRole]=useState('staff')
  const [data,setData]=useState([])
  const [kegiatan,setKegiatan]=useState('')
  const [nominal,setNominal]=useState('')
  const [file,setFile]=useState(null)

  const load = async ()=>{
    const {data} = await supabase.from('csr_proposals').select('*').order('created_at',{ascending:false})
    setData(data||[])
  }
  useEffect(()=>{load()},[])

  const upload = async ()=>{
    if(!kegiatan||!nominal||!file) return alert('Lengkapi!')
    // upload PDF ke storage
    const fileName = Date.now()+'-'+file.name
    await supabase.storage.from('proposal-pdf').upload(fileName, file)
    const {data:{publicUrl}} = supabase.storage.from('proposal-pdf').getPublicUrl(fileName)

    await supabase.from('csr_proposals').insert({
      kegiatan, nominal: parseInt(nominal),
      file_url: publicUrl, file_name: file.name,
      status: 'PENDING_SM'
    })
    setKegiatan(''); setNominal(''); setFile(null); load()
  }

  const approve = async (item)=>{
    let next = null
    if(role==='sm' && item.status==='PENDING_SM') next='PENDING_DIREKTUR'
    if(role==='direktur' && item.status==='PENDING_DIREKTUR') next='BELUM_BAYAR'
    if(role==='finance' && item.status==='BELUM_BAYAR') next='SUDAH_BAYAR'
    if(!next) return alert(`Role ${role} tidak bisa approve status ${item.status}`)
    await supabase.from('csr_proposals').update({status: next}).eq('id', item.id)
    load()
  }

  const laporan = data.filter(d=>d.status==='SUDAH_BAYAR')

  return (
    <div className="p-8 bg-gray-50 min-h-screen">
      <h1 className="text-2xl font-bold mb-4">CSR Approval - SIG (Supabase)</h1>
      <select value={role} onChange={e=>setRole(e.target.value)} className="border p-2 rounded mb-6">
        <option value="staff">Staff Input</option>
        <option value="sm">Senior Manager</option>
        <option value="direktur">Direktur</option>
        <option value="finance">Finance</option>
      </select>

      {role==='staff' && (
        <div className="bg-white p-6 rounded shadow mb-6 flex gap-4">
          <input value={kegiatan} onChange={e=>setKegiatan(e.target.value)} placeholder="Nama Kegiatan" className="border p-2 rounded w-1/3"/>
          <input value={nominal} onChange={e=>setNominal(e.target.value)} type="number" placeholder="Nominal" className="border p-2 rounded"/>
          <input type="file" accept=".pdf" onChange={e=>setFile(e.target.files[0])} className="border p-2 rounded"/>
          <button onClick={upload} className="bg-red-600 text-white px-6 rounded">Upload & Simpan</button>
        </div>
      )}

      <table className="w-full bg-white rounded shadow text-sm">
        <thead className="bg-red-50"><tr><th className="p-3 text-left">Kegiatan</th><th>Nominal</th><th>Status</th><th>PDF</th><th>Aksi</th></tr></thead>
        <tbody>{data.map(d=>(
          <tr key={d.id} className="border-t">
            <td className="p-3">{d.kegiatan}</td>
            <td>Rp {d.nominal?.toLocaleString('id-ID')}</td>
            <td>{d.status}</td>
            <td><a href={d.file_url} target="_blank" className="text-red-600 underline">{d.file_name}</a></td>
            <td><button onClick={()=>approve(d)} className="bg-black text-white px-3 py-1 rounded">Approve</button></td>
          </tr>
        ))}</tbody>
      </table>

      <div className="mt-8 bg-white p-6 rounded shadow">
        <h2 className="font-bold">Laporan CSR (Hanya SUDAH_BAYAR)</h2>
        <p className="text-sm text-gray-500">Total masuk laporan: {laporan.length} kegiatan, Rp {laporan.reduce((a,b)=>a+b.nominal,0).toLocaleString('id-ID')}</p>
      </div>
    </div>
  )
}
